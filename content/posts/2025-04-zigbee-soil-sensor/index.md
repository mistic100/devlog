---
date: '2025-04-05'
title: 'Zigbee soil sensor'
summary: 'Creating a Zigbee device to monitor the soil humidity of multiple house plants.'
tags:
    - ESP32
    - Zigbee
    - Home Assistant
cover:
    image: cover.jpg
---

Since I moved in to my new flat I placed some house plants on a suspended shelf in my living room. I wanted a way to monitor the soil humidity and send alerts to Home Assistant when watering is needed.

There are lot off-the-shelf sensors but they all run on batteries. You also need one for each plant. Creating my own solution was a good small project to experiment with the Zigbee protocol.

{{< image src="images/IMG_20251211_181031.jpg" >}}


## Hardware

The cheapest soil sensor you can find on Aliexpress is the HW-390 capacitive module. It simply needs a 5V power supply and outputs an analog signal depending on the soil humidity.

{{< image src="images/HW-390.jpg" title="HW-390 soil sensor" >}}

For the brain I will use the ESP32-C6 mini developpement board.

{{< image src="images/esp32-c6.jpg" title="ESP32-C6 micro-controller" >}}

I also designed a small 3D printed case to protect the board and have connection pins for multiple sensors.

{{< image src="images/case.png" >}}


## Software

### Sensor calibration

We will read the sensor output with the `analogRead` function which returns a value between `0` and `4095`. But in order to map this value to a humidity level we need to calibrate it.

Here is a simple program to do that.

```c++
void setup()
{
    Serial.begin(115200);
}

void loop()
{
    auto val = analogRead(0);
    Serial.println(val);
    delay(1000);
}
```

Now simply wire a sensor to pin 0 and open the serial monitor to know the value read in different situations. I arbitrarily decided that 0% is when the sensor is in open air and 100% is when the sensor is fully immerged in a glass of water.

### Declare the Zigbee device

The standard Espressif framework already implementations for standard Zigbee devices. It is possible to implement you own but the API and the concepts are a bit difficult to grasp... So I decided to simply exposes multiple `ZigbeeTempSensor` for which I only use the humidity value.

I made a sub-class to make the usage easier.

```c++
class MyZigbeeHumiditySensor : public ZigbeeTempSensor
{
public:
    MyZigbeeHumiditySensor(uint8_t endpoint) : ZigbeeTempSensor(endpoint)
    {
        setManufacturerAndModel("StrangePlanet", "PlantsSensor");
        addHumiditySensor(0, 100, 1);
    }

    void sendHumidity(float value)
    {
        setHumidity(value);
        reportHumidity();
    }
};
```

Then it is only a matter of instanciate it multiple times.

```c++
auto sensor1 = MyZigbeeHumiditySensor(10);
auto sensor2 = MyZigbeeHumiditySensor(11);
auto sensor3 = MyZigbeeHumiditySensor(12);
auto sensor4 = MyZigbeeHumiditySensor(13);

void setup()
{
    Zigbee.addEndpoint(&sensor1);
    Zigbee.addEndpoint(&sensor2);
    Zigbee.addEndpoint(&sensor3);
    Zigbee.addEndpoint(&sensor4);
}
```

Now we need to periodically read each sensors value and send them to the Zigbee endpoints. This can be done in the main `loop` function.

```c++
// values from calibration step
#define MIN_READ 2800
#define MAX_READ 1100

#define ADC_1 0
#define ADC_2 1
#define ADC_3 2
#define ADC_4 3

#define REPORT_TIME_MS 5 * 60 * 1000

long lastReportTime = 0;

void readSensor(MyZigbeeHumiditySensor &sensor, const uint8_t adcPin)
{
    analogRead(adcPin); // the first value can be janky

    auto val = analogRead(adcPin);
    auto humidity = map(val, MIN_READ, MAX_READ, 0, 100);
    humidity = constrain(humidity, 0, 100);

    sensor.sendHumidity(humidity);
}

void loop()
{
    if (millis() - lastReportTime > REPORT_TIME_MS)
    {
        readSensor(sensor1, ADC_1);
        readSensor(sensor2, ADC_2);
        readSensor(sensor3, ADC_3);
        readSensor(sensor4, ADC_4);

        lastReportTime = millis();
    }
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

In order to Z2M to correctly get each sensor data from the device, a custom converter must be declared.

In Home Assistant, custom converters for Z2M are located in your configuration folder in `zigbee2mqtt/external_converters`. Here create a new JavaScript file:

```js
import * as m from 'zigbee-herdsman-converters/lib/modernExtend';

export default {
    zigbeeModel: ['PlantsSensor'],
    model: 'PlantsSensor',
    vendor: 'StrangePlanet',
    description: 'Multi plants soil sensor',
    extend: [
        m.deviceEndpoints({ endpoints: { "1": 10, "2": 11, "3": 12, "4": 13 } }),
        m.humidity({ endpointNames: ["1", "2", "3", "4"] }),
    ],
    meta: { multiEndpoint: true },
};
```

## Home Assistant integration

The end goal was to receive alerts when the soil humidity is too low. There are multiple ways to do that in HA, I choosed to add a task in my todo list.

### Create the task

We will create a new automation which triggers when one of the humidity level is below 50% for 20 minutes (this filters out reading errors).

```yaml
alias: Tâche humidité plantes
triggers:
  - trigger: numeric_state
    entity_id:
      - sensor.plants_sensor_humidity_1
      - sensor.plants_sensor_humidity_2
      - sensor.plants_sensor_humidity_3
      - sensor.plants_sensor_humidity_4
    below: 50
    for:
      hours: 0
      minutes: 20
      seconds: 0
```

Then we create a new item in the todo list, with a condition to not have duplicates.

First define the item name in an script variable.

```yaml
actions:
  - variables:
      label: >-
        Arroser plante {% if trigger.entity_id ==
        'sensor.plants_sensor_humidity_1' %}1{% elif trigger.entity_id ==
        'sensor.plants_sensor_humidity_2' %}2{% elif trigger.entity_id ==
        'sensor.plants_sensor_humidity_3' %}3{% elif trigger.entity_id ==
        'sensor.plants_sensor_humidity_4' %}4{% endif %}
```

Then get all existing items from the list.

```yaml
  - action: todo.get_items
    data:
      status: needs_action
    target:
      entity_id:
        - todo.taches
    response_variable: tasks
```

And finally add the item only if not already present.

```yaml
  - if:
      - condition: template
        value_template: >-
          {{ tasks['todo.taches']['items'] | selectattr('summary', 'eq', label) | list() | count() == 0 }}
    then:
      - action: todo.add_item
        data:
          item: "{{ label }}"
          due_date: "{{ now().strftime('%Y-%m-%d') }}"
        target:
          entity_id:
            - todo.taches
```

### Clear the task

After I watered the plants I want the task to be automatically removed. This can be done with another very similar automation, which triggers when the humidity is above 70% for 10 minutes.

```yaml
alias: Tâche humidité plantes (reset)
triggers:
  - trigger: numeric_state
    entity_id:
      - sensor.plants_sensor_humidity_1
      - sensor.plants_sensor_humidity_2
      - sensor.plants_sensor_humidity_3
      - sensor.plants_sensor_humidity_4
    above: 70
    for:
      hours: 0
      minutes: 10
      seconds: 0
actions:
  - variables:
      label: >-
        Arroser plante {% if trigger.entity_id ==
        'sensor.plants_sensor_humidity_1' %}1{% elif trigger.entity_id ==
        'sensor.plants_sensor_humidity_2' %}2{% elif trigger.entity_id ==
        'sensor.plants_sensor_humidity_3' %}3{% elif trigger.entity_id ==
        'sensor.plants_sensor_humidity_4' %}4{% endif %}
  - action: todo.remove_item
    data:
      item: "{{ label }}"
    target:
      entity_id:
        - todo.taches
```

## Conclusion

If you want to do a similar project you can find all the necessary files on GitHub : [multi-zigbee-soil-sensor](https://github.com/mistic100/multi-zigbee-soil-sensor).

{{< image src="images/IMG_20251211_181046.jpg" >}}
