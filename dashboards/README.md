# Lovelace Dashboard Design

The frontend is managed in **UI/storage mode** (not YAML-mode `ui-lovelace.yaml`), which is why this folder documents the dashboard design rather than shipping a single static config file — the source of truth lives in Home Assistant's `.storage/lovelace.*`, edited live through the UI. The two artifacts below are hand-authored exports that capture the structure and one representative custom card config, sanitized of any personal identifiers.

## Attribution & Customization

The `power-outage-schedule-card` is built on [strange-v/power-outage-schedule-card](https://github.com/strange-v/power-outage-schedule-card). The stock card and its accompanying notification flow were substantially extended for this system:

- **Notification templates rewritten from scratch** (see `notify_power_outage_schedule_added` / `notify_power_outage_schedule_changed_2` in [`/scripts`](../scripts/scripts.yaml)) — the original project doesn't ship a notification pipeline; the red/yellow/green hour-summary push messages, midnight-crossover-aware diffing (only notify on an actual schedule *change*, not every poll), and the actionable-notification prepare/ignore flow are original additions layered on top.
- **Card behavior customized**: the `reload_action` manual-refresh wiring, the `visibility` guard (hides the card instead of rendering broken state before a schedule is published), and the `hide_past_hours` display option were configured/extended specifically for this deployment's UX requirements.

This is a good example of the project's general engineering approach: start from a solid open-source primitive where one exists, then invest the customization effort where it actually matters for the use case (notification logic and graceful degradation) rather than reinventing the base visualization.

## Structure

The dashboard is split into **7 views**, separating the "what do I need to act on right now" surface from deep operational/monitoring surfaces:

| View | Purpose |
|---|---|
| **Home** | Primary control surface — device toggles, the blackout schedule card, EcoFlow status, vacuum map, lighting, weather |
| **Overview** | Secondary rollup |
| **Storage Monitoring** | Router USB storage + disk I/O health (from the OpenWrt telemetry package) |
| **Network Traffic** | WAN/LAN/guest bridge throughput graphs |
| **System Resources** | Router CPU, memory, load average, conntrack table |
| **WiFi & Clients** | Per-radio signal quality, station counts, Starlink failover link |
| **Speedtest & ISP** | Scheduled speedtest history, ISP performance trend |

## Card vocabulary

Card usage is deliberately narrow and consistent rather than mixing many one-off card types — this keeps the visual language predictable across 7 views:

- **`custom:mushroom-template-card` / `mushroom-entity-card` / `mushroom-light-card`** (43 instances) — the primary building block for compact, templated status tiles (used for device toggles, dynamic outage-countdown labels, etc.)
- **`custom:apexcharts-card`** (17 instances) — all time-series graphing (network throughput, CPU/memory trends, speedtest history)
- **`entities` / `horizontal-stack` / `grid`** (41 instances) — layout and grouping primitives
- **`gauge`** (8 instances) — bounded metrics (disk %, CPU %, WiFi signal quality)
- **`custom:power-outage-schedule-card`** (1 instance, see below) — the bespoke blackout-schedule visualization, the centerpiece of the Home view
- **`custom:xiaomi-vacuum-map-card`**, **`custom:clock-weather-card`**, **`custom:mini-graph-card`**, **`picture-entity`**, **`markdown`** — single-purpose cards for the vacuum live map, weather, compact trends, camera snapshot, and free-text notes

## Data density & UX priorities for crisis events

The Home view is designed around the assumption that during a power crisis, the user is glancing at a phone screen for a few seconds, not reading — so the layout front-loads the two questions that matter most:

1. **"Is the power on right now, and for how long?"** — answered instantly by a `mushroom-template-card` pair rendering a Jinja-templated label that flips between *"Until Power ON"* and *"Until Power OFF"* depending on `sensor.power_outage_info.in_outage`, right next to a duration figure. No tapping into history graphs required.
2. **"What's the full schedule for today/tomorrow?"** — answered by the `power-outage-schedule-card`, a purpose-built community card that renders the red/yellow/green hourly blocks from `sensor.power_outage_today_card_display` / `..._tomorrow_card_display` as a visual timeline, with a manual refresh action wired to `homeassistant.update_entity` on the source sensors and a `visibility` guard so the card disappears gracefully instead of showing a broken state when the schedule hasn't been published yet.

Below that, a device grid (power strip, monitor, boiler, kitchen, washing machine, power station inlet) gives one-tap control over exactly the loads that matter for load-shedding decisions, without navigating away from the Home view.

### Example: the outage-schedule card config (sanitized)

```yaml
type: custom:power-outage-schedule-card
queue_entity: input_text.oe_queue
today_entity: sensor.power_outage_today_card_display
tomorrow_entity: sensor.power_outage_tomorrow_card_display
hide_past_hours: true
title: Power outage schedule
empty_text: The schedule for hourly outages will be published by the end of the day.
reload_action:
  service: homeassistant.update_entity
  target:
    - sensor.oe_today
    - sensor.oe_tomorrow
visibility:
  - condition: or
    conditions:
      - condition: and
        conditions:
          - condition: state
            entity: sensor.oe_today
            state_not: unavailable
          - condition: state
            entity: sensor.oe_today
            state_not: unknown
      - condition: and
        conditions:
          - condition: state
            entity: sensor.oe_tomorrow
            state_not: unavailable
          - condition: state
            entity: sensor.oe_tomorrow
            state_not: unknown
```

### Example: templated status tile

```yaml
type: custom:mushroom-template-card
primary: >-
  {% set in_outage = state_attr('sensor.power_outage_info', 'in_outage') %}
  {{ 'Until Power ON' if in_outage else 'Until Power OFF' }}
secondary: "{{ states('sensor.time_until_next_outage') }}"
icon: mdi:power-plug
```

## Resources

Custom cards are pulled via HACS (`/hacsfiles/...` module resources): `mushroom`, `apexcharts-card`, `mini-graph-card`, `button-card`, `banner-card`, `vacuum-card`, `battery-state-card`, `clock-weather-card`, `lovelace-hourly-weather`, `lovelace-auto-entities`, `lovelace-energy-entity-row`, `lovelace-multiple-entity-row`, `lovelace-battery-entity-row`, `lovelace-entity-progress-card`, `config-template-card`, `zigbee2mqtt-networkmap`, and the bespoke `power-outage-schedule-card`.
