# ESP32-CAM AI Water Meter Reader

An automated water meter reading system using an ESP32-CAM, Home Assistant, and AI vision (via LLM Vision + Anthropic Claude). Takes an hourly photo of your water meter, sends it to an AI to read the value, validates the result, and stores it in Home Assistant with full history.

---

## Hardware Required

- ESP32-CAM AI Thinker board
- ESP32-CAM serial programmer board (the small red MB with FTDI chip)
- DS18B20 temperature sensor (optional, for monitoring cabinet temperature)
- 4.7kΩ resistor (for DS18B20 data line pullup)
- USB cable (data capable, not charge-only)

### Wiring — DS18B20

```
VCC  → 3.3V
GND  → GND
DATA → GPIO15 + 4.7kΩ pullup resistor to 3.3V
```
### 3D print recommended - 

Horseshoe-Shaped Universal camera mount for water meters by Zdendys: 

https://www.thingiverse.com/thing:6322373

<img width="1024" height="1024" alt="image" src="https://github.com/user-attachments/assets/4ee917a8-755e-41cf-a799-85dc541b7c8c" />




---

## Software Required

- Home Assistant
- ESPHome (via HA add-on or standalone)
- [LLM Vision](https://github.com/valentinfrlch/ha-llmvision) HACS integration
- Anthropic API key (Claude)

---

## Part 1 — ESPHome Firmware

### Flashing for the First Time

1. Set the IO0→GND jumper on the programmer board
2. Plug in via USB
3. Flash using esptool:

```bash
python -m esptool --chip esp32 --port COM3 --baud 115200 write_flash -z 0x0 firmware.bin
```

Replace `COM3` with your actual port. If it hangs on *Connecting...* press the RST button once while it tries to connect.

4. Remove the IO0 jumper
5. Press RST to boot

### ESPHome YAML Config

Create a new device in ESPHome and paste the following:

```yaml
substitutions:
  devicename: watermeter-camera
  friendly_name: WaterMeter Camera

esphome:
  name: ${devicename}
  friendly_name: ${friendly_name}

esp32:
  board: esp32dev
  framework:
    type: arduino

logger:

api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none
  ap:
    ssid: "${devicename} Fallback"
    password: !secret ap_password

captive_portal:

web_server:
  port: 80

psram:

one_wire:
  - platform: gpio
    pin: GPIO15

esp32_camera:
  id: watermeter_camera
  name: ${friendly_name}
  external_clock:
    pin: GPIO0
    frequency: 20MHz
  i2c_pins:
    sda: GPIO26
    scl: GPIO27
  data_pins: [GPIO5, GPIO18, GPIO19, GPIO21, GPIO36, GPIO39, GPIO34, GPIO35]
  vsync_pin: GPIO25
  href_pin: GPIO23
  pixel_clock_pin: GPIO22
  power_down_pin: GPIO32
  resolution: 800x600
  jpeg_quality: 12
  max_framerate: 10fps
  idle_framerate: 0.2fps
  vertical_flip: false
  horizontal_mirror: false
  aec_mode: auto
  aec2: true
  ae_level: 0
  aec_value: 300
  agc_mode: auto
  agc_gain_ceiling: 2x
  agc_value: 0
  wb_mode: auto

esp32_camera_web_server:
  - port: 8080
    mode: stream
  - port: 8081
    mode: snapshot

sensor:
  - platform: wifi_signal
    name: ${friendly_name} WiFi Signal
    update_interval: 60s
  - platform: uptime
    name: ${friendly_name} Uptime
  - platform: dallas_temp
    name: ${friendly_name} Temperature
    update_interval: 30s

binary_sensor:
  - platform: status
    name: ${friendly_name} Status

output:
  - platform: gpio
    pin: GPIO4
    id: flash_led

light:
  - platform: binary
    output: flash_led
    name: ${friendly_name} Flash LED

switch:
  - platform: restart
    name: ${friendly_name} Restart
```

Add to your `secrets.yaml`:

```yaml
wifi_ssid: "YourSSID"
wifi_password: "YourPassword"
api_key: "generate-in-esphome-dashboard"
ota_password: "your-ota-password"
ap_password: "fallback-password"
```

---

## Part 2 — Home Assistant Setup

### Step 1 — Create Helpers

Go to **Settings → Helpers** and create the following:

| Type | Name | Entity ID | Settings |
|------|------|-----------|----------|
| Number | Water Meter Reading | `input_number.water_meter_reading` | Min: 0, Max: 999999, Step: 0.001, Unit: m³ |
| Number | Water Meter Max Hourly | `input_number.water_meter_max_hourly` | Min: 0, Max: 10, Step: 0.001, Unit: m³, Initial: 1 |
| Number | Water Meter Manual Override | `input_number.water_meter_manual_override` | Min: 0, Max: 999999, Step: 0.001, Unit: m³ |
| Text | Water Meter AI Response | `input_text.water_meter_ai_response` | Max length: 255 |

### Step 2 — Add to configuration.yaml

```yaml
template:
  - sensor:
      - name: Water Meter Reading
        unique_id: water_meter_reading
        unit_of_measurement: m³
        icon: mdi:water
        state: "{{ states('input_number.water_meter_reading') | float }}"
        attributes:
          last_read: "{{ now().strftime('%Y-%m-%d %H:%M') }}"

      - name: Water Meter AI Response
        unique_id: water_meter_ai_response
        icon: mdi:robot
        state: "{{ states('input_text.water_meter_ai_response') }}"

utility_meter:
  water_meter_daily:
    name: Water Daily
    source: sensor.water_meter_reading
    cycle: daily

  water_meter_weekly:
    name: Water Weekly
    source: sensor.water_meter_reading
    cycle: weekly

  water_meter_monthly:
    name: Water Monthly
    source: sensor.water_meter_reading
    cycle: monthly

recorder:
  purge_keep_days: 365
  include:
    entities:
      - input_number.water_meter_reading
      - input_number.water_meter_max_hourly
      - input_number.water_meter_manual_override
      - input_text.water_meter_ai_response
      - sensor.water_meter_reading
      - sensor.water_meter_ai_response
      - sensor.water_meter_daily
      - sensor.water_meter_weekly
      - sensor.water_meter_monthly
```

Restart Home Assistant after saving.

### Step 3 — Set the Initial Meter Reading

Before the automation runs for the first time, set the starting value so validation works correctly:

1. Go to **Developer Tools → States**
2. Find `input_number.water_meter_reading`
3. Set the state to your actual current meter reading (e.g. `1234.567`)
4. Click **Set State**

This gives the validation logic a real baseline to compare against. Without this, every reading will be rejected because the jump from 0 to the real value will exceed the hourly limit.

### Step 4 — Create the Main Automation

Go to **Settings → Automations → Create → Edit in YAML** and paste:

```yaml
alias: Water Meter Reader
description: Takes a photo every hour, reads via LLM Vision, validates and stores result
triggers:
  - trigger: time_pattern
    hours: /1
conditions: []
actions:
  - action: light.turn_on
    target:
      entity_id: light.watermeter_camera_flash_led

  - delay:
      seconds: 2

  - action: camera.snapshot
    target:
      entity_id: camera.watermeter_camera
    data:
      filename: /config/www/watermeter_snapshot.jpg

  - delay:
      seconds: 2

  - action: light.turn_off
    target:
      entity_id: light.watermeter_camera_flash_led

  - action: llmvision.image_analyzer
    data:
      provider: Anthropic
      model: claude-haiku-4-5-20251001
      image_file: /config/www/watermeter_snapshot.jpg
      message: >
        This is an image of a water meter. Read the numeric value shown
        on the meter display. Return ONLY the numeric value as a decimal
        number, nothing else. Example: 1234.567
      max_tokens: 50
    response_variable: ai_response

  - action: logbook.log
    data:
      name: Water Meter
      message: "AI raw response: {{ ai_response.response_text }}"

  - action: input_text.set_value
    target:
      entity_id: input_text.water_meter_ai_response
    data:
      value: "{{ ai_response.response_text }}"

  - variables:
      new_value: "{{ ai_response.response_text | float(0) }}"
      current_value: "{{ states('input_number.water_meter_reading') | float(0) }}"
      max_hourly: "{{ states('input_number.water_meter_max_hourly') | float(1) }}"

  - condition: template
    value_template: "{{ new_value > 0 }}"

  - choose:
      - conditions:
          - condition: template
            value_template: >
              {{ new_value >= current_value and 
                 (new_value - current_value) <= max_hourly }}
        sequence:
          - action: input_number.set_value
            target:
              entity_id: input_number.water_meter_reading
            data:
              value: "{{ new_value }}"
          - action: logbook.log
            data:
              name: Water Meter
              message: >
                Reading ACCEPTED: {{ new_value }} m³ 
                (increase: {{ (new_value - current_value) | round(3) }} m³)

      - conditions:
          - condition: template
            value_template: "{{ new_value < current_value }}"
        sequence:
          - action: logbook.log
            data:
              name: Water Meter
              message: >
                Reading REJECTED (lower than current): AI={{ new_value }} m³, 
                Current={{ current_value }} m³.
          - action: persistent_notification.create
            data:
              title: Water Meter Warning
              message: >
                AI reading {{ new_value }} m³ rejected — lower than current 
                {{ current_value }} m³. Please check camera image.
              notification_id: water_meter_warning

      - conditions:
          - condition: template
            value_template: >
              {{ new_value > current_value and 
                 (new_value - current_value) > max_hourly }}
        sequence:
          - action: logbook.log
            data:
              name: Water Meter
              message: >
                Reading REJECTED (unreasonable jump): AI={{ new_value }} m³, 
                Current={{ current_value }} m³, 
                Increase={{ (new_value - current_value) | round(3) }} m³.
          - action: persistent_notification.create
            data:
              title: Water Meter Warning
              message: >
                AI reading {{ new_value }} m³ rejected — increase of 
                {{ (new_value - current_value) | round(3) }} m³ exceeds 
                hourly limit of {{ max_hourly }} m³.
              notification_id: water_meter_warning

mode: single
```

### Step 5 — Create the Manual Override Automation

```yaml
alias: Water Meter Manual Override
description: Allows manual correction of water meter reading
triggers:
  - trigger: state
    entity_id: input_number.water_meter_manual_override
conditions:
  - condition: template
    value_template: >
      {{ states('input_number.water_meter_manual_override') | float(0) > 0 }}
actions:
  - variables:
      override_value: "{{ states('input_number.water_meter_manual_override') | float }}"
      previous_value: "{{ states('input_number.water_meter_reading') | float }}"

  - action: input_number.set_value
    target:
      entity_id: input_number.water_meter_reading
    data:
      value: "{{ override_value }}"

  - action: logbook.log
    data:
      name: Water Meter
      message: >
        MANUAL OVERRIDE: value changed from {{ previous_value }} m³ 
        to {{ override_value }} m³

  - delay:
      seconds: 2

  - action: input_number.set_value
    target:
      entity_id: input_number.water_meter_manual_override
    data:
      value: 0

  - action: persistent_notification.create
    data:
      title: Water Meter Updated
      message: >
        Reading manually set to {{ override_value }} m³ 
        (was {{ previous_value }} m³)
      notification_id: water_meter_override

mode: single
```

---

## Part 3 — Dashboard

Add a new card in YAML mode:

```yaml
type: vertical-stack
cards:
  - type: entity
    entity: sensor.water_meter_reading
    name: Current Reading
    icon: mdi:water

  - type: statistics-graph
    title: Water Usage History
    entities:
      - sensor.water_meter_reading
    days_to_show: 30
    stat_types:
      - max

  - type: entities
    title: Consumption
    entities:
      - entity: sensor.water_meter_daily
        name: Today
      - entity: sensor.water_meter_weekly
        name: This Week
      - entity: sensor.water_meter_monthly
        name: This Month

  - type: entities
    title: Controls
    entities:
      - entity: input_number.water_meter_max_hourly
        name: Max Hourly Limit (m³)
      - entity: input_number.water_meter_manual_override
        name: Manual Override (set to correct value)

  - type: logbook
    title: Water Meter Log
    entities:
      - input_number.water_meter_reading
      - input_number.water_meter_manual_override
      - sensor.water_meter_reading
      - input_text.water_meter_ai_response
    hours_to_show: 72

  - type: entities
    title: Camera
    entities:
      - entity: sensor.watermeter_camera_temperature
        name: Cabinet Temperature
      - entity: sensor.watermeter_camera_wifi_signal
        name: WiFi Signal
      - entity: light.watermeter_camera_flash_led
        name: Flash LED
      - entity: switch.watermeter_camera_restart
        name: Restart
```

---

## How It All Works

```
ESP32-CAM (hourly trigger)
  → Flash LED on
  → Wait 2 seconds (camera exposure stabilises)
  → Take snapshot → /config/www/watermeter_snapshot.jpg
  → Flash LED off
  → LLM Vision → Anthropic Claude API
  → Validate result:
      ✓ Positive number?
      ✓ Not lower than current reading?
      ✓ Increase within hourly limit?
  → Store in input_number.water_meter_reading
  → Template sensor picks up new value
  → Utility meters calculate daily/weekly/monthly consumption
  → HA Recorder stores full timestamped history
```

---

## Validation Logic

The automation rejects AI readings that are:
- Zero or non-numeric (garbage response)
- Lower than the current stored value (meter can't go backwards)
- More than the max hourly limit above the current value (unreasonable jump)

Rejected readings fire a persistent notification in HA so you can investigate. Tune the **Max Hourly Limit** helper on the dashboard to match your household's realistic maximum consumption.

---

## Correcting a Wrong Reading

Set the correct value in the **Manual Override** slider on the dashboard. The automation picks it up, applies it, logs it, and resets the slider back to zero. The logbook shows a full audit trail of every accepted, rejected, and manually overridden reading.

---

## Tips

- **Prompt tuning** — if the AI consistently misreads certain digits, add more detail to the prompt. For example: "The meter has black digits for whole m³ and red digits for decimals."
- **Camera position** — point directly at the dial face, as close as your lens allows while keeping all digits in frame
- **Lighting** — the onboard flash LED is controlled by the automation. If readings are inconsistent, check the logbook to see what the AI returned each hour
- **First run** — trigger the automation manually via **Developer Tools → Actions** to test it before leaving it to run hourly
- **Max hourly limit** — 1 m³/hr (1000 litres) is a safe default for a house. Lower it if you want tighter validation
