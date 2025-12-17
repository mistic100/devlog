---
date: '2024-12-21'
title: 'Add Wifi to old AC unit'
summary: 'Use ESPHome to make and old AC unit controllable in Home Assistant.'
tags:
    - ESP32
    - Home Assistant
    - ESPHome
cover:
    image: cover.jpg
---

Today we create a device to add connectivity to a rather old AC unit which is only controllable with an infrared remote.

The goal is to build a small box which can emit IR signals and be controlled by Home Assistant. This enables features like power on the unit in heater or cool mode before arriving home, automate power off when leaving home, etc.


## Hardware

In order to send IR signals we need a transmitter capable of 38kHz, these are easy to find and very cheap [on Aliexpress](https://aliexpress.com/item/1005007728215137.html).

{{< image src="images/ir-led.jpg" title="IR LED module" >}}

We also need a micro-controller with Wifi, I choosed the Seeduino ESP32-C3 as I already used in previous projects.

{{< image src="images/esp32-c3.jpg" title="ESP32-C3 micro-controller" >}}

This is absolutely not necessary but I also added a small OLED screen to display the current status of the unit.

{{< image src="images/ssd1306.jpg" title="SSD1306 OLED display" >}}

And a BME280 humidity and temperature sensor to have these informations available in Home Assistant.

{{< image src="images/bme280.jpg" title="BME280 ambient sensor" >}}

Everything is assembled in a 3D printed case sticked with double-sided tape under the AC unit, the LED facing its receiver.

{{< image src="images/render.jpg" size="220" >}} {{< image src="images/IMG_20241221_114847.jpg" size="220" >}}


## Software

The go-to platform to make smart home devices is [ESPHome](https://esphome.io/) which can be installed on a large variety of micro-controllers and offers hundreds of building blocks. ESPHome already has the [IR Remote Climate](https://esphome.io/components/climate/climate_ir/) component which supports a bunch of units, but unfortunately not mine: Fujitsu and Fujitsu General are two different control protocols and ESPHome only supports thr later.

Furtunately there is an Arduino library called [IRremoteESP8266](https://github.com/crankyoldgit/IRremoteESP8266) which supports my unit (and a lot more). All I add to do was to expose the capabilities for IRremoteESP8266 into a ESPHome component. Enters my [ESPHome-IRremoteESP8266](https://github.com/mistic100/ESPHome-IRremoteESP8266) project, which does just that.

Then it is very easy to create a climate device with ESPHome configuration:

```yaml
# declare the board
esphome:
  name: ac-living-room
  friendly_name: ac-living-room

esp32:
  board: seeed_xiao_esp32c3
  variant: esp32c3
  framework:
    type: arduino

logger:

# import the component
external_components:
  - source:
      type: git
      url: https://github.com/mistic100/ESPHome-IRremoteESP8266
    components: [ fujitsu, ir_remote_base ]

# connect to WiFi and enable API
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

api:
  encryption:
    key: <FILL ME>

# declare the IR transmitter
remote_transmitter:
  id: ir_led
  pin: 4
  carrier_duty_percent: 50%

# declare the climate component
climate:
  - platform: fujitsu
    id: clim
    model: ARREB1E
    name: 'Clim'
```

### Temperature sensor integration

A bit more configuration must be added to use the BME280 sensor and reports the data back to Home Assistant:

```yaml
# init I2C controller
i2c:
  sda: 6
  scl: 7
  frequency: 400kHz

# declare the sensor
sensor:
  - platform: bme280_i2c
    address: 0x76
    temperature:
      id: temp_sensor
      name: 'Température'
    pressure:
      id: pres_sensor
      name: 'Pression'
    humidity:
      id: hum_sensor
      name: 'Humidité'

# tie the temperature sensor to the climate controller
climate:
  - platform: fujitsu
    .....
    sensor: temp_sensor
```

### Control the screen

The most difficult part of this project is actually to display something on the screen. Here we leave standard YAML configuration and have to write actual C++ code.

On the screen I show an icon for the current mode, the target temperature and the current temperature and humidity.

{{< image src="images/IMG_20251213_172228.jpg" >}}

```yaml
# load fonts
font:
  - file: 'gfonts://Ubuntu Mono@Bold'
    id: font_large
    size: 40
  - file: 'gfonts://Ubuntu Mono'
    id: font_small
    size: 20

# load icons
image:
  binary:
    chroma_key: 
      - file: 'mdi:fire'
        id: icon_heat
        resize: '48x48'
      - file: 'mdi:snowflake'
        id: icon_cool
        resize: '48x48'
      - file: 'mdi:sun-snowflake-variant'
        id: icon_heat_cool
        resize: '48x48'
      - file: 'mdi:fan'
        id: icon_fan_only
        resize: '48x48'
      - file: 'mdi:water-percent'
        id: icon_dry
        resize: '48x48'
      - file: 'mdi:power'
        id: icon_off
        resize: '48x48'

# declare the screen display
display:
  - platform: ssd1306_i2c
    id: ssd1306_display
    model: 'SSD1306 128x64'
    address: 0x3C
    rotation: '180°'
    contrast: '10%'
    lambda: |-
      auto mode = id(clim).mode;

      if (mode == climate::CLIMATE_MODE_OFF) {
        return;
      }

      // target temperature
      if (mode != climate::CLIMATE_MODE_FAN_ONLY) {
        it.printf(0, 0, id(font_large), "%2.0f°C", id(clim).target_temperature);
      }

      // mode icon
      switch (mode) {
        case climate::CLIMATE_MODE_HEAT:
          it.image(it.get_width()+2, -2, id(icon_heat), ImageAlign::TOP_RIGHT, COLOR_ON, COLOR_OFF);
          break;
        case climate::CLIMATE_MODE_COOL:
          it.image(it.get_width()+2, -2, id(icon_cool), ImageAlign::TOP_RIGHT, COLOR_ON, COLOR_OFF);
          break;
        case climate::CLIMATE_MODE_HEAT_COOL:
          it.image(it.get_width()+2, -2, id(icon_heat_cool), ImageAlign::TOP_RIGHT, COLOR_ON, COLOR_OFF);
          break;
        case climate::CLIMATE_MODE_FAN_ONLY:
          it.image(it.get_width()+2, -2, id(icon_fan_only), ImageAlign::TOP_RIGHT, COLOR_ON, COLOR_OFF);
          break;
        case climate::CLIMATE_MODE_DRY:
          it.image(it.get_width()+2, -2, id(icon_dry), ImageAlign::TOP_RIGHT, COLOR_ON, COLOR_OFF);
          break;
        default:
          it.image(it.get_width()+2, -2, id(icon_off), ImageAlign::TOP_RIGHT, COLOR_ON, COLOR_OFF);
          break;
      }

      it.line(0, 45, it.get_width(), 45);

      // current temperature and humidity
      it.printf(1, it.get_height()+1, id(font_small), TextAlign::BOTTOM_LEFT, "%2.1f°C", id(temp_sensor).state);
      it.printf(it.get_width()-1, it.get_height()+1, id(font_small), TextAlign::BOTTOM_RIGHT, "%3.0f%%", id(hum_sensor).state);
```

As it the screen is always on, which is not always desired, especially if the unit is in your bedroom. That why my actual display goes off when nobody is home, as well as between 23h30 and 9h00, but also goes on for 60 seconds if the settings are changed.

You can find this more complete version here: [ac-control.yaml](files/ac-control.yaml).


## Conclusion

This is a very cheap solution to make any AC unit smart and a good experience with ESPHome.

{{< image src="images/IMG_20241220_184832.jpg" >}}
