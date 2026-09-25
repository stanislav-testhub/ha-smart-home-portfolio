# Blueprints

This system's automations are split between **original, hand-written automations** (see [`/automations`](../automations), [`/scripts`](../scripts), and [`/packages`](../packages)) and a handful of **community blueprints** imported from the Home Assistant Blueprint Exchange for well-solved generic problems (battery alerts, motion lighting, etc.).

In line with attribution norms for shared blueprints, the third-party YAML files themselves are not vendored into this repository — only the blueprints actually used are listed below, with the automation that consumes them.

| Blueprint | Author | Used for |
|---|---|---|
| `low-battery-level-detection-notification-for-all-battery-sensors.yaml` | [sbyx](https://github.com/sbyx) | Fleet-wide low-battery push notifications across all Zigbee/BLE sensors |
| `zigbee2mqtt-tuya-moes-smart-knob-ers-10tzbvk-aa.yaml` | [rdeangel](https://github.com/rdeangel) | Smart knob → light brightness/toggle mapping |
| `appliance-notifications.yaml`, `battery-charger-notifications.yaml`, `sensor-light.yaml` | [Blackshome](https://github.com/Blackshome) | Appliance/charger state notifications, sensor-driven lighting |
| `appliance-status-monitor.yaml` | [leofabri](https://github.com/leofabri) | Generic appliance run/finish monitoring |
| `presence_non_binary.yaml` | [cliffordwhansen](https://github.com/cliffordwhansen) | Presence detection from non-binary occupancy sensors |
| `reload-integrations.en.yaml` | [danimart1991](https://github.com/danimart1991) | Auto-reload of flaky integrations |
| `auto_update_scheduled.yaml` | [edwardtfn](https://github.com/edwardtfn) | Scheduled update installs |
| `esphome_auto_update_after_addon.yaml` | [zenguru84](https://github.com/zenguru84) | ESPHome firmware auto-update |
| `event_summary.yaml` | [valentinfrlch](https://github.com/valentinfrlch) | Daily/periodic event digest |
| `actionable-notifications-for-android.yaml`, `send-camera-snapshot-notification-on-motion.yaml` | [vorion](https://github.com/vorion) | Actionable push notifications, camera snapshot on motion |
| `motion_light.yaml`, `notify_leaving_zone.yaml` | Home Assistant core blueprints | Motion-activated lighting, zone-leave notifications |
| `confirmable_notification.yaml` | Home Assistant core blueprints (script) | Two-way confirmable push notification |
| `inverted_binary_sensor.yaml` | Home Assistant core blueprints (template) | Inverted binary sensor helper |

The two most business-critical blueprint integrations — the smart knob and the low-battery watchdog — are wired directly into `automations.yaml` (see `Smart bulb control` and `Low battery level detection & notification for all battery sensors`), demonstrating how third-party blueprints are composed with original automation logic rather than used as a substitute for it.
