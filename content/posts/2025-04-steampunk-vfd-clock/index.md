---
date: '2025-04-19'
title: 'Steampunk VFD clock'
summary: 'Making a steampunk themed clock using the IV-27 VFD tube.'
tags:
    - ESP32
    - Clock
cover:
    image: cover.jpg
---

I already made [two clocks](https://photos.strangeplanet.fr/index.php?/category/73-horloge_nixie_in_14_avril_2012_juin_2020) using [Nixie tubes](https://photos.strangeplanet.fr/index.php?/category/219-horloge_nixie_in_12_aout_2021). I love them but they can only display digits, I wanted something a bit more advanced capable of displaying basic text like the full date or even messages.

To stay in the same aesthetic I used the IV-27 VFD (Vacuum Fluorescent Display) tube which as 13 nice green 7 segments characters.

Two articles helped me to understand how to drive the tube:

- [VFD Alarm Clock instructable by ChristineNZ](https://www.instructables.com/VFD-Alarm-Clock/)
- [V-27M IceTube Clock Project by Barbouri](https://www.barbouri.com/2020/07/04/iv-27-icetube-clock-project/)


## Hardware

The IV-27 tube is a multiplexed display: it has 8 inputs for each part of a digit (7 segments + dot) and 14 grids to select which digit to display. Thats 22 pins to drive at 25 volts.

{{< image src="images/iv-27.jpg" title="IV-27 VFD tube" >}}

For this job we use the MAX6921 IC which has 20 high-voltage outputs (I choose to not use the first grid which only has symbols nor the right-most digit).
The MAX6921 is a simple serial to parallel converter: to send a data frame we have to put DIN low or high, ticking CLK for each value, once the 20 bits are send, ticking LOAD sets the outputs accordingly.

Because the display is multiplexed we have to repetitively set each digit in an infinite loop, as fast as possible.

{{< image src="images/MAX6921.jpg" title="MAX6921 serial to parallel converter" >}}

As I didn't want to order a custom PCB and do all my assembly on a proto-board, I ordered this SOP28 to DIP adapter.

{{< image src="images/sop28-adapter.jpg" title="SOP28 to DIP adapter" >}}

The brain of the clock will a ESP32-C3 micro controller with its Wifi antenna.

{{< image src="images/esp32-c3.jpg" title="ESP32-C3 micro-controller" >}}

Other components include:

- a BME280 sensor for temperature and humidity reading
- an optional Real Time Clock module (though I didn't used it because I rely on the Network Time Protocol)
- two DC-DC power converters
    - a 5V to 24V boost converter for the display anode (through the MAX6921)
    - a 5V to 3.3V buck converter for the display grid
- a rotary encoder
- two LED noodles

{{< image size="175" src="images/bme280.jpg" title="BME280 ambient sensor" >}}
{{< image size="175" src="images/ds3231.jpg" title="DS3231 RTC module" >}}
{{< image size="175" src="images/rotary-encoder.jpg" >}}
{{< image size="175" src="images/led-noodle.jpg" >}}

### PCB

There are not so much components so the wiring is rather simple. The most cumbersome part is the 22 wires between the MAX6921 and the tube, half on each side.

You will notice the use of a transitor to drive the LED from the micro controller, and capacitors to filter out the rotary encoder signals.

{{< image src="images/schematics.png" size="500" >}}

As I put everything on a protoboard, my assembly is very ugly... but it works!

{{< image src="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184309-c90fa0d0-xs.jpg" href="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184309-c90fa0d0-xl.jpg" >}} {{< image src="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184310-105d4c1c-xs.jpg" href="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184310-105d4c1c-xl.jpg" >}}

Even more when all wires are in place.

{{< image src="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184312-0ec058ac-xs.jpg" href="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184312-0ec058ac-xl.jpg" >}}

### Case design

I mentionned I wanted a nice steampunk look to my clock, that means brass, copper and valves!

The tube is mounted to the main case with standard plumbing fittings.

{{< image src="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184312-de2b1992-xs.jpg" href="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184312-de2b1992-xl.jpg" >}}

The case itself is 3D printed with two coats of copper effect paint.

{{< image src="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184308-f4dc60c7-xs.jpg" href="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184308-f4dc60c7-xl.jpg" >}}

A red valve is fitted to the rotary encoder and LED noodles are glued bellow the tube.

{{< image src="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184313-20e437fc-xs.jpg" href="https://photos.strangeplanet.fr/_data/i/upload/2025/04/07/20250407184313-20e437fc-xl.jpg" >}}


## Software

I won't dive to deep into the firmware as there is a lot going on, everything can be found on [my GitHub repository](https://github.com/mistic100/IV-27-Clock).

The features of this clock are:

- get date and time from NTP or the RTC module
- get temperature and humidity from the BME sensor
- have different "screens" selected with the rotary encoder
    - time
    - date
    - temperature
- automatically turn off at night
- optionally display short messages from Home Assistant
- have some settings for date format, light intensity, etc
- flash the firmware over-the-air (OTA)

---

I tried to apply some Object Oriented Programming principles and isolate responsibilities in classes:

- `DateTimeWrapper`: a class to abstract the usage of NTP or the RTC module
- `Settings`: stores and gets settings in the non-volatile memory
- `HaSensor`: gets data from Home Assistant using HTTP
- `Display`: controls the VFD tube through the serial to parallel converter
- `Light`: controls the lights
- `Controller`: the main class responsible to query data and tell the display what to show, it is basically a state machine
- `Ui`: responds to inputs ooff the rotary encoder to the change the controller state

### Home Assistant

The HA integration consists of knowing if there is someone at home and fetch the message to display. For that I made a custom sensor that gets the nextdue item in my to-do list:

```yaml
template:
  - triggers:
      - trigger: state
        entity_id:
          - todo.my_list
          - zone.home
        # scheduler to handle item status change
      - trigger: time_pattern
        minutes: "/5"
    action:
      - service: todo.get_items
        data:
          status: needs_action
        target:
          entity_id: todo.my_list
        response_variable: items
    binary_sensor:
      - unique_id: data_for_iv27_clock
        state: true
        attributes:
          at_home: "{{ states('zone.home') | int(default=0) > 0 }}"
          message: >
            {% set tdate = (now().date() + timedelta(days=1)) | string %}
            {{ items['todo.my_list']['items'] 
              | selectattr('due', 'defined') | selectattr('due', 'lt', tdate) 
              | map(attribute='summary') | first() | truncate(255) }}
```

## Conclusion

If you want to do a similar project you can find all the necessary files and installation instructions on GitHub : [IV-27-Clock](https://github.com/mistic100/IV-27-Clock).
