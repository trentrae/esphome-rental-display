# ESPHome Rental Property Display

A touchscreen dashboard for rental properties built with [ESPHome](https://esphome.io) and [LVGL](https://lvgl.io). Runs on the **Waveshare ESP32-S3-Touch-LCD-7B** (1024×600, capacitive touch) and connects to [Home Assistant](https://www.home-assistant.io/) for real-time status and control.

Designed for short-term rental (Airbnb/VRBO) hosts who manage multiple properties — deploy to a new property by changing a single `substitutions.yaml` file.

---

## Features

| Page | Description |
|------|-------------|
| **Main Status** | Outside/inside temp, front & back door lock status, south & north garage status, security summary, 5-day weather forecast, quick actions (all lights off, garage toggle), current time |
| **Security** | Alarmo integration with full keypad — arm away/home/night/vacation/custom, disarm with code, countdown timer, sensor status list |
| **Locks & Doors** | Front/back door lock cards (tap to lock, long-press to unlock), door contact sensor status, garage door card with toggle |
| **Lighting** | Individual toggle cards for 3 lights, all-on / all-off quick actions |
| **Access Logs** | Placeholder page for future guest access logging |

### Additional Capabilities

- **Screen saver** — dims backlight after 5 minutes of inactivity, touch to wake
- **Event-driven updates** — all entity states pushed via `on_value`/`on_state` callbacks (no polling)
- **Dark theme** — iOS-inspired dark palette optimized for always-on wall displays
- **OTA updates** — over-the-air flashing with LVGL pause during transfer

---

## Hardware Requirements

| Component | Specification |
|-----------|---------------|
| **Display** | [Waveshare ESP32-S3-Touch-LCD-7B](https://www.waveshare.com/esp32-s3-touch-lcd-7b.htm) |
| **SoC** | ESP32-S3-N16R8 (16 MB flash, 8 MB PSRAM) |
| **Resolution** | 1024 × 600 RGB parallel |
| **Touch** | GT911 capacitive (I2C) |
| **IO Expander** | CH32V003 (backlight PWM, reset pins) |

> **Note:** This project is specifically designed for the 7B model. Other Waveshare displays have different pin mappings and resolutions.

---

## Software Requirements

| Software | Version |
|----------|---------|
| **ESPHome** | 2026.2.0+ |
| **Home Assistant** | 2024.1+ |
| **Framework** | ESP-IDF (not Arduino) |

### Home Assistant Integrations (Required)

| Integration | Purpose |
|-------------|---------|
| [Alarmo](https://github.com/nielsfaber/alarmo) | Alarm panel with arm modes and countdown |
| Z-Wave / Zigbee locks | Lock/unlock status and control |
| Konnected (or any binary sensor) | Door contact and garage sensors |
| Weather integration | 5-day forecast via template sensors |

---

## Installation

### Option A: Remote Package (Recommended)

Pull configuration directly from this GitHub repo. ESPHome fetches and caches the files automatically.

**1. Create your local ESPHome config file** (e.g., `esp32-7-display.yaml`):

```yaml
esphome:
  name: rental-display-7b
  friendly_name: "Rental Property Display 7B"

packages:
  - url: https://github.com/trentrae/esphome-rental-display
    ref: main
    files:
      - waveshare7b/device.yaml
      - waveshare7b/common.yaml
      - waveshare7b/substitutions.yaml
      - waveshare7b/layouts/fonts.yaml
      - waveshare7b/layouts/colors.yaml
      - waveshare7b/layouts/main.yaml
      - waveshare7b/layouts/update_intervals_pkg.yaml
    refresh: 0s

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "Rental Display Fallback"
    password: !secret wifi_password

api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password
    on_begin:
      then:
        - lvgl.pause:

logger:
  level: INFO

web_server:
  port: 80
```

**2. Create `substitutions.yaml`** in this repo (see [Setup: Substitutions](#setup-substitutions) below).

**3. Create `secrets.yaml`** with your WiFi and API credentials (see `secrets.yaml.example`).

**4. Flash:**

```bash
esphome run esp32-7-display.yaml
```

### Option B: Local Clone

Clone the repo directly into your ESPHome config directory.

```bash
cd /config/esphome
git clone https://github.com/trentrae/esphome-rental-display.git
```

Then use the included `waveshare7b/index.yaml` as your entry point:

```bash
esphome run esphome-rental-display/waveshare7b/index.yaml
```

> **Note:** You still need to create `substitutions.yaml` and `secrets.yaml` inside the `waveshare7b/` directory.

---

## Setup: Substitutions

The `substitutions.yaml` file is the **only file you need to change** per property. It maps your Home Assistant entity IDs to the display.

**1. Copy the example:**

```bash
cp waveshare7b/substitutions.yaml.example waveshare7b/substitutions.yaml
```

**2. Edit `substitutions.yaml`** and replace every entity ID with the correct one from your HA instance:

```yaml
substitutions:
  # Property name shown in header
  property_name: "Beach House"

  # Alarm panel (Alarmo)
  alarm_panel: "alarm_control_panel.alarmo"

  # Locks (must be lock.* entities — sends "locked"/"unlocked" strings)
  front_door_lock: "lock.front_door_deadbolt"
  back_door_lock: "lock.patio_door"

  # Door contact sensors (binary_sensor — on/off)
  front_door_sensor: "binary_sensor.front_door"
  back_door_sensor: "binary_sensor.back_door"

  # Garage sensors
  south_garage_sensor: "cover.garage_door"          # cover.* entities use string states
  north_garage_sensor: "binary_sensor.north_garage"  # binary_sensor.* entities use on/off

  # Garage covers (for tap-to-toggle control)
  south_garage_door: "cover.garage_door"
  north_garage_door: "cover.north_garage_door"

  # Lights (can be light.* or switch.* — uses homeassistant.turn_on/off)
  living_room_light: "light.living_room"
  kitchen_light: "light.kitchen"
  outdoor_light: "switch.porch_light"

  # Temperature sensors
  outside_temp: "sensor.outside_temperature"
  inside_temp: "sensor.inside_temperature"

  # Weather entity (for 5-day forecast template sensors)
  weather_entity: "weather.home"
```

### Entity Type Reference

| Substitution | Expected Entity Domain | State Format |
|---|---|---|
| `front_door_lock`, `back_door_lock` | `lock.*` | `"locked"` / `"unlocked"` (string) |
| `south_garage_sensor` | `cover.*` or `binary_sensor.*` | `"open"` / `"closed"` (string) or on/off |
| `north_garage_sensor` | `binary_sensor.*` | on / off (boolean) |
| `front_door_sensor`, `back_door_sensor` | `binary_sensor.*` | on / off (boolean) |
| `living_room_light`, `kitchen_light`, `outdoor_light` | `light.*` or `switch.*` | on / off (boolean) |
| `alarm_panel` | `alarm_control_panel.*` | `"disarmed"`, `"armed_away"`, etc. (string) |
| `outside_temp`, `inside_temp` | `sensor.*` | numeric (°F) |

---

## Setup: Weather Forecast

The 5-day weather forecast requires **20 template sensors** in Home Assistant. These extract forecast data from your weather integration and expose it as individual sensors that ESPHome can subscribe to.

### Create Template Sensors in HA

Add the following to your Home Assistant `configuration.yaml` (or a `template.yaml` package):

```yaml
template:
  - trigger:
      - platform: time_pattern
        hours: "/1"    # Update every hour
      - platform: homeassistant
        event: start
    action:
      - service: weather.get_forecasts
        target:
          entity_id: weather.YOUR_WEATHER_ENTITY
        data:
          type: daily
        response_variable: forecast
    sensor:
      # ── High Temperatures ──
      - name: "Weather Forecast High 0"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[0].temperature }}"
        unit_of_measurement: "°F"
      - name: "Weather Forecast High 1"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[1].temperature }}"
        unit_of_measurement: "°F"
      - name: "Weather Forecast High 2"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[2].temperature }}"
        unit_of_measurement: "°F"
      - name: "Weather Forecast High 3"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[3].temperature }}"
        unit_of_measurement: "°F"
      - name: "Weather Forecast High 4"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[4].temperature }}"
        unit_of_measurement: "°F"

      # ── Low Temperatures ──
      - name: "Weather Forecast Low 0"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[0].templow }}"
        unit_of_measurement: "°F"
      - name: "Weather Forecast Low 1"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[1].templow }}"
        unit_of_measurement: "°F"
      - name: "Weather Forecast Low 2"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[2].templow }}"
        unit_of_measurement: "°F"
      - name: "Weather Forecast Low 3"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[3].templow }}"
        unit_of_measurement: "°F"
      - name: "Weather Forecast Low 4"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[4].templow }}"
        unit_of_measurement: "°F"

      # ── Conditions ──
      - name: "Weather Forecast Condition 0"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[0].condition }}"
      - name: "Weather Forecast Condition 1"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[1].condition }}"
      - name: "Weather Forecast Condition 2"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[2].condition }}"
      - name: "Weather Forecast Condition 3"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[3].condition }}"
      - name: "Weather Forecast Condition 4"
        state: "{{ forecast['weather.YOUR_WEATHER_ENTITY'].forecast[4].condition }}"

      # ── Day Names ──
      - name: "Weather Forecast Day 0"
        state: "{{ as_timestamp(forecast['weather.YOUR_WEATHER_ENTITY'].forecast[0].datetime) | timestamp_custom('%a') }}"
      - name: "Weather Forecast Day 1"
        state: "{{ as_timestamp(forecast['weather.YOUR_WEATHER_ENTITY'].forecast[1].datetime) | timestamp_custom('%a') }}"
      - name: "Weather Forecast Day 2"
        state: "{{ as_timestamp(forecast['weather.YOUR_WEATHER_ENTITY'].forecast[2].datetime) | timestamp_custom('%a') }}"
      - name: "Weather Forecast Day 3"
        state: "{{ as_timestamp(forecast['weather.YOUR_WEATHER_ENTITY'].forecast[3].datetime) | timestamp_custom('%a') }}"
      - name: "Weather Forecast Day 4"
        state: "{{ as_timestamp(forecast['weather.YOUR_WEATHER_ENTITY'].forecast[4].datetime) | timestamp_custom('%a') }}"
```

> Replace `weather.YOUR_WEATHER_ENTITY` with your actual weather entity ID (e.g., `weather.home`, `weather.ktki_daynight`).

After adding, restart Home Assistant and verify the sensors appear in **Developer Tools → States**.

---

## Project Structure

```
waveshare7b/
├── index.yaml                  # Local entry point (direct flash)
├── _package.yaml               # Remote package reference docs
├── device.yaml                 # Hardware: pins, display, touch, backlight, IO expander
├── common.yaml                 # HA sensors, scripts, globals, event-driven callbacks
├── substitutions.yaml          # YOUR entity IDs (git-ignored, create from .example)
├── substitutions.yaml.example  # Template with all required substitutions
├── secrets.yaml.example        # Template for WiFi/API/OTA credentials
└── layouts/
    ├── main.yaml               # LVGL orchestrator: pages, theme, touchscreen binding
    ├── fonts.yaml              # Roboto + Material Design Icons (7 text sizes, 4 icon sizes)
    ├── colors.yaml             # Dark theme color palette
    ├── update_intervals.yaml   # Polling: alarm countdown (1s), WiFi/clock (30s)
    └── pages/
        ├── main_status.yaml    # Dashboard: temps, locks, garages, weather, quick actions
        ├── alarm_control.yaml  # Alarmo keypad: 6 arm modes, code entry, countdown
        ├── locks_doors.yaml    # Lock/door detail cards with tap-to-control
        ├── lighting_control.yaml  # 3 light toggles + all-on/all-off
        └── access_logs.yaml    # Placeholder for future guest log page
```

### File Responsibilities

| File | What It Controls |
|------|-----------------|
| `device.yaml` | GPIO pin assignments, RGB display timings, GT911 touch config, CH32V003 IO expander, backlight PWM. **Never edit unless changing hardware.** |
| `common.yaml` | All Home Assistant entity bindings (`sensor`, `text_sensor`, `binary_sensor`), event-driven LVGL update callbacks, alarm scripts, security status logic, screen-saver timer. |
| `substitutions.yaml` | Entity ID mapping — the **only file that changes per property**. |
| `layouts/main.yaml` | LVGL page declarations, theme defaults, display/touch bindings, update interval. |
| `layouts/fonts.yaml` | Font definitions: Roboto (12–36px) and MDI icons (28, 36, 48, 56px). |
| `layouts/pages/*.yaml` | Individual page layouts — pure LVGL widget trees. |

---

## Deploying to a New Property

This project is designed for multi-property deployment. Each property gets its own ESPHome device with a unique `substitutions.yaml`.

### Step-by-Step

1. **Set up the new property's Home Assistant instance** with:
   - Alarmo alarm panel
   - Lock entities (Z-Wave, Zigbee, etc.)
   - Door contact sensors
   - Garage door sensors/covers
   - Temperature sensors
   - Weather integration
   - 20 weather forecast template sensors (see [Setup: Weather Forecast](#setup-weather-forecast))

2. **Copy `substitutions.yaml.example`** to `substitutions.yaml`

3. **Update every entity ID** to match the new HA instance:
   ```yaml
   substitutions:
     property_name: "Mountain Cabin"
     front_door_lock: "lock.cabin_front"
     # ... etc
   ```

4. **Update `secrets.yaml`** with the property's WiFi credentials and API keys

5. **Flash the device:**
   ```bash
   esphome run esp32-7-display.yaml
   ```

6. **Add the device in Home Assistant** → Settings → Devices & Services → ESPHome

### Finding Your Entity IDs

In Home Assistant:
- **Developer Tools → States** — search for your entities
- **Settings → Devices** — click a device to see all its entities
- Entity IDs follow the pattern `domain.name` (e.g., `lock.front_door`, `sensor.outside_temperature`)

---

## Adding or Changing Devices

### Adding a New Sensor to the Display

**Example:** Add a pool temperature sensor to the main status page.

1. **Add the substitution** in `substitutions.yaml.example` and your `substitutions.yaml`:
   ```yaml
   pool_temp: "sensor.pool_temperature"
   ```

2. **Add the HA sensor binding** in `common.yaml` under the appropriate section:
   ```yaml
   sensor:
     # ... existing sensors ...
     - platform: homeassistant
       id: pool_temp_sensor
       entity_id: ${pool_temp}
       internal: true
       on_value:
         - lambda: |-
             char buf[10]; snprintf(buf, sizeof(buf), "%.0f°F", x);
             lv_label_set_text(id(pool_temp_label), buf);
   ```

3. **Add the LVGL widget** in the appropriate page file (e.g., `layouts/pages/main_status.yaml`):
   ```yaml
   - label:
       id: pool_temp_label
       text: "--°F"
       text_font: roboto_32
       text_color: 0x0A84FF
       align: CENTER
   ```

### Adding a New Light Toggle

1. **Add the substitution:**
   ```yaml
   patio_light: "light.patio"
   ```

2. **Add the binary sensor** in `common.yaml` under `binary_sensor:`:
   ```yaml
   - platform: homeassistant
     id: patio_light_state
     entity_id: ${patio_light}
     internal: true
     on_state:
       - lambda: |-
           lv_color_t c = x ? lv_color_hex(0xFFD700) : lv_color_hex(0x8E8E93);
           lv_label_set_text(id(patio_toggle_text), x ? "ON" : "OFF");
           lv_obj_set_style_text_color(id(patio_toggle_text), c, 0);
           lv_obj_set_style_text_color(id(patio_light_icon), c, 0);
   ```

3. **Add the light card** in `layouts/pages/lighting_control.yaml` (copy an existing card and update IDs).

4. **Add the toggle action** to the card's `on_click`:
   ```yaml
   on_click:
     - homeassistant.service:
         service: homeassistant.toggle
         data:
           entity_id: ${patio_light}
   ```

### Adding a New Lock

1. **Add substitution:**
   ```yaml
   side_door_lock: "lock.side_door"
   ```

2. **Add in `common.yaml`** under `text_sensor:` (locks use string states, not binary):
   ```yaml
   - platform: homeassistant
     id: side_door_locked
     entity_id: ${side_door_lock}
     internal: true
     on_value:
       - lambda: |-
           std::string s = x;
           for (auto &ch : s) ch = tolower(ch);
           bool locked = (s == "locked");
           lv_color_t c = locked ? lv_color_hex(0x30D158) : lv_color_hex(0xFF453A);
           const char* t = locked ? "Locked" : "Unlocked";
           const char* icon = locked ? "\U000F033E" : "\U000F033F";
           lv_label_set_text(id(side_door_status), t);
           lv_obj_set_style_text_color(id(side_door_status), c, 0);
           lv_label_set_text(id(side_door_icon), icon);
           lv_obj_set_style_text_color(id(side_door_icon), c, 0);
       - script.execute: update_security_status
   ```

3. **Add widget** in the appropriate page layout.

4. **Update `update_security_status`** script in `common.yaml` to include the new lock in the security check.

### Changing an Entity Type

**Important:** Different HA entity domains require different ESPHome sensor platforms:

| Entity Domain | ESPHome Platform | State Type | Callback |
|---|---|---|---|
| `lock.*` | `text_sensor` | `"locked"` / `"unlocked"` | `on_value` (string `x`) |
| `cover.*` | `text_sensor` | `"open"` / `"closed"` | `on_value` (string `x`) |
| `binary_sensor.*` | `binary_sensor` | `true` / `false` | `on_state` (bool `x`) |
| `light.*`, `switch.*` | `binary_sensor` | `true` / `false` | `on_state` (bool `x`) |
| `sensor.*` | `sensor` | numeric | `on_value` (float `x`) |
| `alarm_control_panel.*` | `text_sensor` | `"disarmed"`, `"armed_away"`, etc. | `on_value` (string `x`) |

> **Common gotcha:** If you see `Can't convert 'locked' to binary state!` in the logs, the entity needs to be a `text_sensor` instead of `binary_sensor`.

### Adding a New Page

1. **Create the page layout** in `layouts/pages/your_page.yaml`

2. **Register it** in `layouts/main.yaml`:
   ```yaml
   pages:
     # ... existing pages ...
     - id: page_your_page
       widgets: !include pages/your_page.yaml
   ```

3. **Add a nav button** — copy an existing nav button from any page and update the icon and `lvgl.page.show` target.

4. **Add any required HA entity bindings** in `common.yaml`.

---

## Architecture Notes

### Event-Driven Updates

All Home Assistant entity updates flow through ESPHome's native API subscription — **no polling**. Each entity has an `on_value` or `on_state` callback that directly calls LVGL C functions (`lv_label_set_text`, `lv_obj_set_style_text_color`, etc.) to update the display instantly.

The only polling is:
- **1-second interval** — alarm countdown timer (only active during `pending` state)
- **30-second interval** — WiFi signal bars and clock (no HA callback for these)
- **500ms LVGL render loop** — redraws dirty widgets

### LVGL Widget IDs

Widgets are referenced by `id` across files. The naming convention is:

```
{entity}_{property}_{page}
```

Examples:
- `front_door_icon_main` — front door icon on the main status page
- `front_door_icon_locks` — front door icon on the locks & doors page
- `garage_status_main` — garage status text on the main status page

### Fonts

| Font ID | Size | Usage |
|---------|------|-------|
| `roboto_12` – `roboto_36` | 12–36px | UI text at various sizes |
| `mdi_28` | 28px | Navigation bar icons |
| `mdi_36` | 36px | Compact card icons (lock cards on main page) |
| `mdi_48` | 48px | Standard card icons |
| `mdi_56` | 56px | Headline icons (alarm page) |

To add a new MDI icon, find its codepoint at [pictogrammers.com/library/mdi](https://pictogrammers.com/library/mdi/) and add it to the appropriate font's `glyphs` list in `fonts.yaml`:

```yaml
- "\U000FXXXX"  # icon-name
```

### Color Palette

| Color | Hex | Usage |
|-------|-----|-------|
| Background Primary | `#1C1C1E` | Page background |
| Background Secondary | `#2C2C2E` | Cards |
| Background Tertiary | `#3A3A3C` | Elevated elements |
| Green | `#30D158` | Locked, closed, secured, ON |
| Red | `#FF453A` | Unlocked, triggered, errors |
| Orange | `#FF9F0A` | Garage open, arming |
| Yellow | `#FFD700` | Lights on |
| Blue | `#0A84FF` | Accent, active buttons |
| Gray | `#8E8E93` | Inactive, disabled |

---

## Troubleshooting

### `Can't convert 'locked' to binary state!`
The lock entity is bound as a `binary_sensor` but needs to be a `text_sensor`. Lock entities send string states (`"locked"`/`"unlocked"`), not boolean on/off.

### `Can't convert 'closed' to binary state!`
Same issue but for a `cover.*` entity. Cover entities send `"open"`/`"closed"` strings. Use `text_sensor` with string comparison.

### Glyph not found (`lv_draw_sw_letter: glyph dsc. not found for U+XXXX`)
The MDI icon codepoint isn't included in the font being used. Either:
- Add the glyph to the correct font in `fonts.yaml`
- Change the label's `text_font` to a font that includes the glyph

### Display stays black after boot
The CH32V003 IO expander controls the backlight. Ensure:
- `external_components` with `github://pr#10071` is present in `device.yaml`
- The `exio2` switch has `restore_mode: ALWAYS_ON`

### Lock status not updating
Verify in the ESPHome logs:
1. The lock appears as `Homeassistant Text Sensor` (not `Binary Sensor`) in the config dump
2. You see `[lock] Front door state raw: 'unlocked'` debug messages
3. There is only **one** `text_sensor:` top-level key in `common.yaml` (duplicate YAML keys cause silent data loss)

### ESPHome not picking up changes from GitHub
ESPHome caches remote packages. Set `refresh: 0s` in your local yaml to force re-download on every compile. The log message `Reverting changes to ... -> <hash>` is normal — it means the cache is being reset to the latest commit.

---

## License

MIT

---

## Credits

- [ESPHome](https://esphome.io) — firmware framework
- [LVGL](https://lvgl.io) — graphics library
- [Alarmo](https://github.com/nielsfaber/alarmo) — alarm panel integration
- [Material Design Icons](https://pictogrammers.com/library/mdi/) — UI icons
- [Roboto](https://fonts.google.com/specimen/Roboto) — UI font
- Display pin mapping reference: [agillis/esphome-modular-lvgl-buttons](https://github.com/agillis/esphome-modular-lvgl-buttons)
