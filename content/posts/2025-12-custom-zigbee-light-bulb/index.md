---
date: '2025-12-09'
title: 'Custom Zigbee light bulb'
summary: 'My short journey to create a better light bulb for sunrise effect.'
tags:
    - ESP32
    - Zigbee
    - Home Assistant
    - FastLED
cover:
    image: cover.jpg
---

A while ago a bought a generic Zigbee light bulb ([this one](https://www.zigbee2mqtt.io/devices/IC-CDZFB2AC004HA-MZN.html)) with the intention to use it in my bedroom as a cheap sunrise simulator. So far it works very well, and it had helped me wake up more easily.

The only complain I have about this light (and as far as I know, every Zigbee light bulb on the market) is that the minimum brightness is not so minimal. It happens that I wake immediately as it lights up at minimum brightness. So I decided to make my own!

The goal is to use the Zigbee capability of some ESP32 chips to control an adressable LED strip.


## Hardware

For the hardware I am working with a ESP32-C6 mini developpement board. It is also possible to use the ESP32-H2 which is a bit cheaper has it does not have WiFi (the C6 has WiFi + Bluetooth + Zigbee).

> **Note:** some ESP32-C6 have a battery LED indicator which stays on or blinks when no battery is connected. To disable it you can solder a 10kΩ resistor between the BAT+ pad and the 5V pin.

{{< image src="images/esp32-c6.jpg" title="ESP32-C6 micro-controller" >}}

I also use a SK6812 WWA addressable LED strip I had laying around. The WWA variant is a nice little thing which is controlled like a WS2812 strip, but instead of Red-Green-Blue channels you have Amber-Warm White-Cold White.

{{< image src="images/sk6812-wwa.jpg" title="SK6812 WWA LED strip" >}}

For the power part I am using a E14 to E27 light bulb adapter from which I stripped the inner part and connected a small 5V AC-DC converter.

> **Obligatory warning:** live wires are dangerous, be safe!

{{< image src="images/e14-e27-adapter.jpg" title="E14 to E27 adapter" >}}
{{< image src="images/e14-e27-adapter-2.jpg" title="Removed the inner part and soldered wires" >}}


### Case

In order to contain the controller, the power adapter and the LED strip, and secured everything to the socket I designed a 3D printable case in SolidWorks.

The main body is basically a tube in which is packed the electronics, and the strip is wrapped around. It screws to the socket with the original screws. And a lid closes everything.

Finally a transparent cover printed in vase mode goes over everything, its purpose is not to diffuse light but only to protect the LED strip from dust.

{{< image src="images/body.jpg" title="Main body" >}}
{{< image src="images/assembly.jpg" title="Assembly with socket and electronics" >}}

{{< image src="images/IMG_20251217_183756.jpg" >}}
{{< image src="images/IMG_20251217_190758.jpg" >}}
{{< image src="images/IMG_20251217_190856.jpg" >}}


## Software

I am using PlatformIO extension on Visual Studio Code. Before starting to code, here is the `platformio.ini` and `partitions.csv` files needed to compile.

```ini
[env:esp32-c6-devkitm-1]
platform = https://github.com/pioarduino/platform-espressif32/releases/download/stable/platform-espressif32.zip
board = esp32-c6-devkitm-1
framework = arduino
board_build.partitions = partitions.csv
build_flags = 
  -DARDUINO_USB_CDC_ON_BOOT=1
  -DARDUINO_USB_MODE=1
  -DCORE_DEBUG_LEVEL=3
  -DZIGBEE_MODE_ED
monitor_speed = 115200
lib_deps =
  FastLED
```

```csv
# Name,     Type, SubType, Offset,  Size, Flags
nvs,        data, nvs,     0x9000,  0x5000,
otadata,    data, ota,     0xe000,  0x2000,
app0,       app,  ota_0,   0x10000, 0x140000,
app1,       app,  ota_1,   0x150000,0x140000,
spiffs,     data, spiffs,  0x290000,0x15B000,
zb_storage, data, fat,     0x3EB000,0x4000,
zb_fct,     data, fat,     0x3EF000,0x1000,
coredump,   data, coredump,0x3F0000,0x10000,
```

### Declare the Zigbee light

We will use the `ZigbeeColorDimmableLight` class which allows to declare a light with changeable brightness and temperature.

_Note:_ as of writing, the arduino-esp32 framework has not yet been released with temperature support, [this pull request](https://github.com/espressif/arduino-esp32/pull/12094) having been merged only recently. It will probably be available in version 3.3.5.

```c++
auto light = ZigbeeColorDimmableLight(10);

void setup()
{
    light.setManufacturerAndModel("StrangePlanet", "DimLight");
    light.setLightColorCapabilities(ZIGBEE_COLOR_CAPABILITY_COLOR_TEMP);
    light.onLightChangeTemp(setLight);

    Zigbee.addEndpoint(&light);
}
```

The `setLight` callback is called when the state of the light changes. To control the strip I use the FastLED library, this will allow to add effects in the future! But first we need to the declare the strip.

```c++
#define LED_PIN 0
#define NUM_LEDS 52

CRGB leds[NUM_LEDS];

void setup()
{
    FastLED.addLeds<WS2812, LED_PIN, BRG>(leds, NUM_LEDS);
    FastLED.setCorrection(UncorrectedColor);

    setLight(false, 255, ESP_ZB_ZCL_COLOR_CONTROL_COLOR_TEMPERATURE_DEF_VALUE);
}

void loop()
{
    FastLED.show();
}
```

Now lets implement the `setLight` function. The goal is to map the temperature value provided by the Zigbee controller in mired unit (with Kelvin = 1.000.000 / mired) to "fake" RGB values with the following mapping:

- Red → Amber (~1800K)
- Green → Warm White (~3000K)
- Blue → Cold White (~6000K)

```c++
#define TEMP_AMBER 1800
#define TEMP_WARM 3000
#define TEMP_COLD 6000

void setLight(bool on, uint8_t level, uint16_t mireds)
{
    if (!on || level == 0)
    {
        FastLED.setBrightness(0);
        return;
    }

    CRGB color;

    uint16_t temp = constrain(1000000.0 / mireds, TEMP_AMBER, TEMP_COLD);

    if (temp <= TEMP_WARM)
    {
        // fade amber to warm
        auto factor = (uint8_t) map(temp, TEMP_AMBER, TEMP_WARM, 0, 255);
        color.r = 255 - factor;
        color.g = factor;
        color.b = 0;
    }
    else
    {
        // fade warm to cold
        auto factor = (uint8_t) map(temp, TEMP_WARM, TEMP_COLD, 0, 255);
        color.r = 0;
        color.g = 255 - factor;
        color.b = factor;
    }

    fill_solid(leds, NUM_LEDS, color);
    FastLED.setBrightness(level);
}
```

### Connect to a Zigbee network

The final step is to connect to a Zigbee network on boot. It is a very crude implementation and lacks at least a way to reset the registered network, for example with a button press or a boot counter.

```c++
void setup()
{
    Serial.begin(115200);

    if (!Zigbee.begin())
    {
        Serial.println("Zigbee failed to start!");
        delay(1000);
        esp_restart();
    }
    else
    {
        Serial.println("Zigbee started successfully!");
    }

    Serial.print("Connecting to network ");
    while (!Zigbee.connected())
    {
        Serial.print(".");
        delay(100);
    }

    Serial.println();
    Serial.println("Ready");
}
```

### Configure Zigbee2Mqtt

This step is optional as Z2M will automatically detects the device capabilities. However it is better to register a custom converter to fine tune the device configuration.

In Home Assistant, custom converters for Z2M are located in your configuration folder in `zigbee2mqtt/external_converters`. Here create a new JavaScript file:

```js
import * as m from 'zigbee-herdsman-converters/lib/modernExtend';

export default {
    zigbeeModel: ['DimLight'],
    model: 'DimLight',
    vendor: 'StrangePlanet',
    description: 'DimLight',
    extend: [
        m.light({
            effect: false,
            powerOnBehavior: false,
            colorTemp: {
                startup: false,
                range: [166, 555], // this is the 1800-6000K range converted to mired
            },
        }),
    ],
};
```

And restart Z2M to load the new configuration for the device.

### Run the sunrise effect

In order to animate the light from dim orange to bright white I use this [Light Transition Effect](https://github.com/Mirai-Miki/home-assistant-blueprints/blob/main/ha_light_transition_blueprint.yaml) blueprint in Home Assistant. The script looks like this:

```yaml
alias: Sunrise chambre (script)
use_blueprint:
  path: Mirai-Miki/ha_light_transition_blueprint.yaml
  input:
    light_entity: light.lampadaire_chambre
    color_mode: Temperature
    initial_temp: 1800
    target_temp: 4000
    transition_time:
      hours: 0
      minutes: 20
      seconds: 0
    target_brightness: 80
```


## Conclusion

Now I have a custom light bulb which can be lit very dim and progressively increase the brightness and temperature to wake me up nicely.

This photo show the difference of brightness at minimum for the commercial bulb (bottom) and my version (top). Concurrently the commercial version has a higher maximum brightness.

{{< image src="images/IMG_20251217_191252.jpg" >}}

If you want to create a similar device you can find here all my work files:

- SolidWorks model
- PlatformIO project
- Z2M custom converter
  
[zigbee-light-bulb (dec 2025).zip - 8 Mio](files/zigbee-light-bulb%20(dec%202025).zip)
