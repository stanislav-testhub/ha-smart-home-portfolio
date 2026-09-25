# Home Assistant Smart Home Automation Platform

[![CI](https://github.com/stanislav-testhub/ha-smart-home-portfolio/actions/workflows/ci.yml/badge.svg)](https://github.com/stanislav-testhub/ha-smart-home-portfolio/actions/workflows/ci.yml)

A self-hosted **Home Assistant OS** deployment engineered around a real, high-stakes constraint: an electricity grid subject to scheduled rolling blackouts. This repository is a sanitized export of a production configuration, showcasing automation architecture, third-party API integration, defensive engineering patterns, and QA discipline applied to home automation — not a toy demo.

> **Anonymization notice** — All personally identifying data (GPS coordinates, home address, external domain, internal IP addresses, device/entity registry IDs, MAC addresses, Zigbee IEEE addresses, and the account holder's name) has been replaced with descriptive `<PLACEHOLDER>` tokens or generic slugs (e.g. `lan_ip_3`) before publication. Business logic, entity naming conventions, and automation structure are otherwise unmodified from the live system.

## Repository Layout

```
├── configuration/        # Core configuration.yaml (recorder, HTTP hardening, helpers)
├── automations/          # automations.yaml — general-purpose home automations
├── scripts/               # scripts.yaml — reusable notification scripts
├── packages/              # Self-contained HA "packages" (feature-scoped config bundles)
│   ├── electricity_outage_schedule.yaml   # Blackout schedule ingestion + crisis management
│   ├── vacuum_cleaner_scheduler.yaml      # Presence-aware robot vacuum scheduling
│   └── openwrt_collectd.yaml              # Router telemetry via MQTT/collectd
├── blueprints/            # Attribution list for community blueprints in use
└── dashboards/            # Lovelace UI structure, card vocabulary, sanitized card configs
```

## 1. Architecture Overview

The deployment runs **Home Assistant OS** under the Supervisor, with configuration split using HA's [packages](https://www.home-assistant.io/docs/configuration/packages/) feature (`homeassistant: packages: !include_dir_named packages`) so that each cross-cutting feature — blackout management, vacuum scheduling, router telemetry — owns its own helpers, template sensors, and automations in one file instead of being scattered across the global `automations.yaml` / `sensor:` blocks. This is a deliberate architectural choice: it keeps feature boundaries legible and makes each package independently portable/removable.

**Integration surface:**

| Layer | Technology |
|---|---|
| Zigbee | Zigbee2MQTT add-on, bridged over MQTT (Mosquitto add-on) |
| Xiaomi ecosystem | `xiaomi_miot` / `xiaomi_home` (robot vacuum, smart plugs), `miwifi` (router) |
| Firmware nodes | ESPHome-flashed devices, managed via the ESPHome add-on |
| Network hardware | OpenWrt router, telemetry ingested via `collectd`'s `exec`/`network` plugin over MQTT — **one-way**: the router publishes, Home Assistant only subscribes and never commands it back. Companion repo: [openwrt-router-portfolio](https://github.com/stanislav-testhub/openwrt-router-portfolio) |
| Frontend | HACS-managed custom Lovelace cards (Mushroom, ApexCharts, Mini Graph, custom outage-schedule card) |
| External integrations | REST polling of a regional energy-distributor API; Healthchecks.io dead-man's-switch monitoring |
| Persistence | SQLite recorder, tuned (`purge_keep_days: 7`, `commit_interval: 20s`) to bound disk I/O and DB growth on constrained storage |

**Security hardening baked into `configuration.yaml`:**
- `http.trusted_proxies` scoped to the Supervisor's internal Docker network only (not `0.0.0.0/0`), with `use_x_forwarded_for` enabled for correct client-IP logging behind the Supervisor's proxy.
- `ip_ban_enabled` + `login_attempts_threshold: 2`, **layered with a custom automation** (`Ban Suspicious IP`) that parses `system_log_event` warnings for failed logins, regex-extracts the offending IP, cross-checks a whitelist helper, and calls `http.ban_ip` — effectively a home-grown fail2ban implemented declaratively in YAML/Jinja, with its own audit trail via `persistent_notification.create` and `system_log.write`.
- A `rest_command` heartbeat pings an external Healthchecks.io endpoint every 3 minutes — a synthetic dead-man's-switch so that a hung or crashed HA process is detected externally, independent of HA's own alerting (which would obviously be unavailable if HA itself is down).

## 2. Blackout & Power Crisis Management System

The centerpiece of the system: `packages/electricity_outage_schedule.yaml`. The region's grid operator publishes rolling blackout schedules ("queues") per address; this package turns that public schedule into a fully automated preparation → mitigation → recovery pipeline.

### Data ingestion
Two `rest:` sensors poll the utility's public JSON API:
- **`schedule-by-search`** (2h interval) resolves the address to its blackout "queue" group.
- **`schedule-by-queue`** (30 min interval) pulls the actual hour-by-hour schedule for today and tomorrow for that queue.

Requests carry realistic browser headers (`User-Agent`, `Accept-Language`, `Referer`, etc.) — a pragmatic resilience choice to avoid the integration breaking against naive bot-filtering on a third-party API never designed for machine consumption.

### Normalization layer
Rather than letting every downstream automation re-parse the raw `HH:MM-HH:MM-status;...` schedule strings, a single template sensor (`Power Outage Info`) does it once: it parses both days' schedules, **merges a today/tomorrow outage block across the midnight boundary** if they're contiguous, and emits one structured JSON payload (`in_outage`, `time_until_minutes`, `next_duration_hours`, `next_range`, `next_day`). Every notification, automation, and dashboard card reads this single source of truth — eliminating duplicated parsing logic and the drift/bugs that come with it.

### Staged, human-in-the-loop notifications
Warnings fire at 30-minute and ~3-4 minute windows before an outage, each gated by an **exact single-minute template condition** (e.g. `<=30 and >29`) against a `time_pattern: minutes: '*'` trigger — so each threshold fires exactly once, not once per minute for the whole window. Notifications use iOS actionable-notification buttons (`TURN_OFF_DEVICES` / `TURN_ON_DEVICES` / `IGNORE`), putting a human in the loop for actions with real consequences instead of silently draining a battery backup on every schedule change.

### Defensive device actuation
The backup power station's primary switch is unreliable to command directly in the field, so the automation drives a secondary smart relay in a `repeat: ... until:` loop that watches the actual device state rather than firing a single blind command — a retry pattern purpose-built around a known-flaky Zigbee/Xiaomi device. Every step that waits on a device confirming a state change uses `wait_for_trigger` with an explicit `timeout:` and `continue_on_timeout: true`, so the flow always completes (and notifies) instead of hanging if a device doesn't respond.

### Recovery detection with independent verification
Power restoration is detected **two independent ways**: from the schedule prediction, and separately from a real illuminance sensor debounced into `binary_sensor.stable_light_state` (`delay_on: 40s`, `delay_off: 2min`). Scheduled outages routinely end early or run long in practice, so relying on the published schedule alone would be wrong; cross-checking against physical ground truth is the safety net (see [§5 Race Conditions](#race-conditions-grid-flickering) for how flicker is specifically handled).

## 3. Smart Energy Management

- **Dynamic tariff tracking**: `input_number.electricity_rate` (₴/kWh, updated as rates change) feeds the built-in Home Assistant **Energy Dashboard** as the price source for multiple tracked circuits, rather than a hardcoded cost constant.
- **Five independently metered grid sources** feed the Energy Dashboard: main smart-relay energy, kitchen socket, washing machine, power strip, and the backup power station's inlet — each priced either from the rate helper directly or from a re-exposed sensor wrapping it, demonstrating both of HA's supported Energy Dashboard pricing mechanisms.
- **Per-session energy accounting for the backup power station**: because the station's cumulative energy sensor is monotonically increasing over its lifetime (not per-charge-session), an automation snapshots the counter into `input_number.power_station_energy_start` every time the inlet switch turns on, and a template sensor computes `current − start` — the only way to answer "what did *this specific outage* cost in backup energy" without a hardware counter reset.
- **Current-sensing appliance-state inference**: the washing machine's smart switch doesn't expose a "cycle finished" state, so a `numeric_state` trigger watches `sensor.washing_machine_current` staying below 0.35A for 3 minutes to infer completion — with a live "on duration" template sensor computed directly from `last_changed` for the dashboard.
- **Silent-failure detection via duration accumulation**: `Power Zero Duration` is a trigger-based template sensor that accumulates "hours since last non-zero power reading" entirely in Jinja state (no `python_script`, no extra helper), used to flag a load that's gone silently dead (e.g. a tripped breaker) rather than just being switched off.
- **Time-of-day-aware load shedding**: 25-30 minutes before a scheduled outage, non-critical loads under a power threshold are proactively cut, and the power station's AC/DC output auto-off timeout is switched between a longer daytime profile ("2 hr") and a conservative overnight profile ("30 min") — balancing inverter idle-drain against typical outage duration by time of day.

## 4. Lovelace Dashboard Design

See [`/dashboards`](./dashboards) for the full breakdown (7 views, card vocabulary, sanitized card configs, and attribution/customization notes for the outage-schedule card). In short: the **Home** view is designed for a 3-second glance during a crisis — a templated status tile answers "is power on, and for how long" before anything else renders, backed by a purpose-built `power-outage-schedule-card` rendering the full red/yellow/green hourly timeline, with a manual-refresh action and a `visibility` guard that hides the card gracefully instead of showing broken state when the schedule hasn't been published yet. Six secondary views (Storage, Network Traffic, System Resources, WiFi & Clients, Speedtest & ISP) surface the OpenWrt telemetry package for deep network operations visibility, kept separate from the crisis-response surface so the two use cases don't compete for screen space.

## 5. QA, Validation & Edge Cases

Home Assistant automations don't run in a conventional CI/unit-test harness, so validation is built into the architecture itself rather than bolted on afterward:

- **Isolating testable logic**: all non-trivial parsing/business logic (schedule parsing, midnight-crossover merging, duration formatting) lives in **template sensors**, not inline inside action blocks — this means it can be exercised and inspected independently in HA's Template Developer Tools against captured real API payloads, without needing to trigger a full automation run.
- **Mocking a real-world event that can't be scheduled on demand**: a grid outage can't be triggered for a test run. The pipeline is validated instead by injecting synthetic values into `sensor.oe_today` / `sensor.oe_tomorrow` via Developer Tools → States, in the exact `DD.MM.YYYY;<approved-since>;HH:MM-HH:MM-status;...` format the REST integration itself parses. Because every downstream sensor (`Power Outage Info`, the two card-display sensors, the dashboard card) is driven purely by state changes on those two entities rather than by the poll itself, this exercises the *entire* parsing → aggregation → dashboard chain end-to-end without waiting for a real blackout — the same technique used to produce the dashboard screenshots for this README.
- **Entity health linting**: [Watchman](https://github.com/dummylabs/thewatchman) (`.storage/watchman.stats`) runs as a continuous check for broken entity references and dangling automations — functionally the closest equivalent to a static-analysis pass over YAML-defined automation logic, where a typo'd `entity_id` would otherwise fail silently.
- **External synthetic monitoring**: the 3-minute Healthchecks.io heartbeat is a smoke test for the HA process itself — if HA hangs, crashes, or loses network, alerting fires from *outside* the system, since HA's own notification pipeline would obviously be unavailable in that failure mode.

### Race conditions ("grid flickering")
Flickering grid power (very short on/off transitions right at an outage boundary) is a known failure mode for naive automations. This system handles it with layered defenses:
- The physical power-restoration signal (`stable_light_state`) is **debounced asymmetrically** — 40s to confirm "on", 2 minutes to confirm "off" — so a brief flicker can't toggle it, and the "on" side is intentionally faster than "off" since a false-positive "power's back" is lower-cost than a false-negative during a real restoration.
- The triggering automation **re-validates the raw sensor value inside its own condition block** at execution time, not just at trigger time — guarding against the classic race where the triggering state changes again in the gap between trigger and action execution.
- Threshold notifications use an **exact single-minute window condition** against a per-minute trigger, rather than a `for:` duration on a derived value — because the derived "minutes until outage" value legitimately changes every single minute regardless of anything "settling," a `for:` condition would either never satisfy or fire on the wrong minute.
- `mode: single` is used deliberately on state-mutating crisis-management automations (to guarantee no concurrent re-entry mid-sequence), while `mode: queued` (`max: 10`) is used on general device-control automations where safely queuing multiple trigger events is the correct behavior — the mode is chosen per automation based on whether concurrent execution is dangerous, not defaulted blindly.

### Failure modes: integrations and devices going offline
- **`availability:` templates** are explicitly attached to computed sensors (e.g. session energy, zero-power duration) so they surface as `unavailable` rather than silently freezing at a stale value or defaulting to `0` when their upstream source entity disappears — preventing an integration outage from being misread as "no power draw."
- A dedicated **`Power Sensor Status Monitor`** automation distinguishes "sensor unresponsive for 5 minutes" from "load reads zero" as two different alerts — so a dead Zigbee end device isn't misdiagnosed as "the appliance is off."
- Every wait on a physical device confirming a command (`wait_for_trigger`) carries an explicit `timeout:` with `continue_on_timeout: true`, so a non-responsive smart plug can't stall an entire crisis-response sequence — the flow always completes and notifies.
- **Capped-retry circuit breaker** for host reboots: the network-loss self-heal automation reboots at most 3 times, then backs off for 30 minutes and resets its counter — preventing a reboot crash-loop if the actual fault is upstream (e.g. the router itself, not Home Assistant).
- The **Zigbee2MQTT bridge watchdog** detects the bridge switch reporting "off" for 3+ minutes and retries a restart every 5 minutes for as long as it stays down — automated recovery from the most common self-hosted Zigbee failure mode (the coordinator/bridge process silently dying) without manual intervention.

## Continuous validation

Every push runs [`yamllint`](https://yamllint.readthedocs.io/) across `automations/`, `configuration/`, `packages/`, and `scripts/`, plus a [gitleaks](https://github.com/gitleaks/gitleaks) secret scan — see [`.github/workflows/ci.yml`](.github/workflows/ci.yml). The same "validate, don't assume" discipline described in [§5](#5-qa-validation--edge-cases) applied to the repo itself, not just the automations.

---

*This repository documents architecture and automation logic. It is not intended to be deployed as-is — device IDs, entity IDs, and network details are placeholders (see anonymization notice above) and must be replaced with your own hardware's identifiers.*
