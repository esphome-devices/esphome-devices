---
title: Blitzwolf BW-SHP6 Socket with powermonitoring
date-published: 2020-08-06
type: plug
standard: eu
---
  ![Product](./BlitzWolf-BW-SHP6.jpg "Product Image")

Model reference: BW-SHP6

Manufacturer: [BlitzWolf](https://www.blitzwolfeurope.com/3840W-EU-WIFI-Smart-Socket-BlitzWolf-BW-SHP2-Wifi)

There are two version of this plug, a 10A version and a 15A version. The pinout seems to be the same.

## GPIO Pinout

| Pin    | Function (<2020>)          | Function (>2020)           |
|--------|----------------------------|----------------------------|
| GPIO13 | Button  (inverted)         | Button  (inverted)         |
| GPIO00 | Red LED (inverted)         | Red LED (inverted)         |
| GPIO15 | Relay                      | Relay                      |
| GPIO02 | Blue LED (inverted)        | Blue LED (inverted)        |
| GPIO12 | HLW8012 - SEL              | HLW8012 - SEL              |
| GPIO05 | HLW8012 - CF               | HLW8012 - CF               |
| GPIO14 | HLW8012 - CF1              |                            |
| GPIO04 |                            | HLW8012 - CF1              |

## HLW8012 Calibration Values

| Value   | <2020   | >2020   |
|---------|---------|---------|
| Current | 0.00290 | 0.00117 |
| Voltage | 940     | 755     |

## Basic Config

The configuration has some default sensors for wifi reporting etc.

```yaml
substitutions:
  device_name: shp6
  # Higher value gives lower watt readout
  current_res: '0.00290'
  # Lower value gives lower voltage readout
  voltage_div: '940'
  # 2020 model uses GPIO04 for CF1
  cf1_pin: GPIO14
  # BW-SHP6, outlet with powermonitoring.
  # One button for the relay, and one red led for the relay, as well as a blue status led
  # Static IP is configured, and fast_connect is enabled, as the SSID is hidden
  # Webserver is active and pw protected, and the OTA is password protected

esphome:
  name: '${device_name}'
  platform: ESP8266
  board: esp8285
  on_boot:
    then:
      - switch.turn_on: relay

wifi:
  ssid: !secret wifissid
  password: !secret wifipw

logger:
  baud_rate: 0

api:

ota:
  password: !secret otapw

# Button configuration
binary_sensor:
  - platform: gpio
    name: "${device_name} button"
    id: button
    pin:
      number: GPIO13
      inverted: true
    on_press:
      then:
        - switch.toggle: relay

# Setup of LED's used in displaying Switch status
output:
  - platform: gpio
    pin: GPIO00
    inverted: true
    id: led

# Config for switch
switch:
  - platform: gpio
    pin: GPIO15
    restore_mode: RESTORE_DEFAULT_OFF
    id: relay
    name: '${device_name} Relay'
    on_turn_on:
      - output.turn_on: led
    on_turn_off:
      - output.turn_off: led

# Status LED for connection
status_led:
  pin:
    number: GPIO02
    inverted: true

# Sensors for Voltage (V), Current (A), Power (kW), Daily energy usage (kWh)
sensor:
  - platform: hlw8012
    sel_pin:
      number: GPIO12
      inverted: true
    cf_pin: GPIO05
    cf1_pin: ${cf1_pin}
    current_resistor: ${current_res}
    voltage_divider: ${voltage_div}
    current:
      name: '${device_name} Current'
      unit_of_measurement: 'A'
      accuracy_decimals: 3
      icon: mdi:flash-outline
    voltage:
      name: '${device_name} Voltage'
      unit_of_measurement: 'V'
      icon: mdi:flash-outline
    power:
      name: '${device_name} Power'
      unit_of_measurement: 'kW'
      accuracy_decimals: 3
      id: power
      filters:
        - multiply: 0.001
      icon: mdi:flash-outline
    change_mode_every: 4
    update_interval: 10s
  - platform: total_daily_energy
    name: '${device_name} daily energy'
    power_id: power
    unit_of_measurement: 'kWh'
    accuracy_decimals: 5
```

## Advanced config additions

Under wifi this can be added, this will set up static IP, allow the device to connect to a hidden SSID (fast_connect) and create a backup AP

```yaml
wifi:
  reboot_timeout: 60min
  manual_ip:
    static_ip: 192.168.1.100
    gateway: 192.168.1.1
    subnet: 255.255.255.0
  fast_connect: true

# Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "${device_name} Hotspot"
    password: !secret appw
```

This wll activate the internal webserver with password protection

```yaml
web_server:
  port: 80
  auth:
    username: !secret webuser
    password: !secret webpw
```

To set time locally to the same as on the HomeAssistant (better logging)

```yaml
# Sets time from Homeassistant
time:
  - platform: homeassistant
    id: homeassistant_time
```

To have different data shown for the device (ESPHome version) and the wifi. Will appear as sensors in HA.

```yaml
# Sensors for ESP version and WIFI information
text_sensor:
  - platform: version
    name: "${device_name} ESPHome Version"
  - platform: wifi_info
    ip_address:
      name: "${device_name} ip"
    ssid:
      name: "${device_name} ssid"
```

This will create sensors so that you can track wifi coverage for the devices, and also note the uptime for the devices.

```yaml
sensors:
  - platform: wifi_signal
    name: '${device_name} WiFi Signal'
    update_interval: 60s
    accuracy_decimals: 0
  - platform: uptime
    name: '${device_name} Uptime'
    unit_of_measurement: days
    update_interval: 300s
    accuracy_decimals: 1
    filters:
      - multiply: 0.000011574
```
