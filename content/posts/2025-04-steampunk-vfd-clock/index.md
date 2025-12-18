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

Because the display is multiplexed we have to repetitively set each digit in an infinite loop, as fast as possible. And as I didn't want to order a custom PCB and do all my assembly on a proto-board, I ordered this SOP28 to DIP adapter.

{{< image src="images/MAX6921.jpg" title="MAX6921 serial to parallel converter soldered to its SOP28 adapter" >}}

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

I won't dive too deep into the firmware as there is a lot going on, everything can be found on [my GitHub repository](https://github.com/mistic100/IV-27-Clock).

The features of this clock are:

- get date and time from NTP or the RTC module
- get temperature and humidity from the BME sensor
- have different "screens" selected with the rotary encoder
    - time
    - date
    - temperature
    - settings
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
- `Ui`: responds to inputs of the rotary encoder to the change the controller state

### Display control

The `Display#loop` method is worth detailing, it is responsible of lighting up the right segments in the tube for the text we want to display.

#### Basic principle

As mentionned earlier the tube has 14 grids (think "character select") and 8 segments (standard 7 segments + dot), each one of these inputs (only 12 grids actually) are assigned to an output of the serial-to-parallel converter. The converter is controlled by three pins: CLK, DIN and LOAD.

Because I soldered wires ramdomly between the converter and the tube I have two mapping table between the desired grid/segment and the corresponding output:

```c++
uint8_t GRID[NUM_GRIDS] = {
    // 14 unused
    2,  // 13
    18, // 12
    1,  // 11
    17, // 10
    0,  // 9
    16, // 8
    5,  // 7
    7,  // 6
    6,  // 5
    4,  // 4
    19, // 3
    3,  // 2
    // 1 unused
};

uint8_t SEGMENTS[8] = {
    10, // A
    12, // B
    15, // C
    9,  // D
    14, // E
    11, // F
    13, // G
    8,  // Pt
};
```

To light up a particular segment in a particular grid, we need to set high the corresponding pins, that means set DIN 20 times to high or low state, "ticking" CLK for every bit, then tick LOAD to "commit" the data. For example to light up the segment A of the first grid a basic code would be like:

```c++
// initialize data
byte data = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};
data[GRID[0]] = 1;
data[SEGMENTS[0]] = 1;

// loop each bit
for (int k = 19; k >= 0; k--) {
    // set DIN
    digitalWrite(DRIVER_DIN, data[k]);
    // tick CLK
    digitalWrite(DRIVER_CLK, HIGH);
    digitalWrite(DRIVER_CLK, LOW);
}
// tick LOAD
digitalWrite(DRIVER_LOAD, HIGH);
digitalWrite(DRIVER_LOAD, LOW);
```

Now the outputs 2 and 10 of the converter are high, and if the wiring is correct, one segment is on in the tube.

#### More characters

To scale up things we need a way to convert characters (0 to 9 and A to Z) into arrays of segments to light up. Lets introduce two new mapping tables:

```c++
// ASCII 48-57
std::array<byte, 7> SEGMENTS_NUMS[] = {
    {1, 1, 1, 1, 1, 1, 0}, // 0
    {0, 1, 1, 0, 0, 0, 0}, // 1
    {1, 1, 0, 1, 1, 0, 1}, // 2
    {1, 1, 1, 1, 0, 0, 1}, // 3
    {0, 1, 1, 0, 0, 1, 1}, // 4
    {1, 0, 1, 1, 0, 1, 1}, // 5
    {1, 0, 1, 1, 1, 1, 1}, // 6
    {1, 1, 1, 0, 0, 0, 0}, // 7
    {1, 1, 1, 1, 1, 1, 1}, // 8
    {1, 1, 1, 0, 0, 1, 1}, // 9
};

// ASCII 65-90
std::array<byte, 7> SEGMENTS_ALPHA[] = {
    {1, 1, 1, 0, 1, 1, 1}, // A
    {0, 0, 1, 1, 1, 1, 1}, // b
    {1, 0, 0, 1, 1, 1, 0}, // C
    {0, 1, 1, 1, 1, 0, 1}, // d
    {1, 0, 0, 1, 1, 1, 1}, // E
    {1, 0, 0, 0, 1, 1, 1}, // F
    {1, 0, 1, 1, 1, 1, 0}, // G
    {0, 1, 1, 0, 1, 1, 1}, // H
    {0, 1, 1, 0, 0, 0, 0}, // I
    {0, 1, 1, 1, 1, 0, 0}, // J
    {0, 1, 0, 1, 1, 1, 1}, // K
    {0, 0, 0, 1, 1, 1, 0}, // L
    {1, 1, 0, 1, 0, 1, 0}, // M
    {1, 1, 1, 0, 1, 1, 0}, // N
    {1, 1, 1, 1, 1, 1, 0}, // O
    {1, 1, 0, 0, 1, 1, 1}, // P
    {1, 1, 0, 1, 0, 1, 1}, // Q
    {1, 1, 0, 0, 1, 1, 0}, // R
    {1, 0, 1, 1, 0, 1, 1}, // S
    {0, 0, 0, 1, 1, 1, 1}, // t
    {0, 1, 1, 1, 1, 1, 0}, // U
    {0, 1, 0, 0, 0, 1, 1}, // v
    {0, 1, 1, 1, 1, 1, 1}, // W
    {0, 0, 1, 0, 0, 1, 1}, // X
    {0, 1, 1, 1, 0, 1, 1}, // Y
    {1, 1, 0, 1, 1, 0, 1}, // Z
};

std::array<byte, 7> SEGMENTS_BLANK = {0, 0, 0, 0, 0, 0, 0};
```

Lets also declare a string variable which contains the desired text to display (it is managed by the `Controller` class). And a function which will update the `data` array with the correct bits for a particular character in this string.

```c++
char chars[NUM_GRIDS];

// Updates `data` 8 segments for current character
void setChar(byte index) {
    const char s = chars[index];

    // it is a number
    if (s >= 48 && s <= 57) {
        setChar(SEGMENTS_NUMS[s - 48]);
    }
    // it is an uppercase letter
    else if (s >= 65 && s <= 90)  {
        setChar(SEGMENTS_ALPHA[s - 65]);
    }
    // it is a lowercase letter
    else if (s >= 97 && s <= 122) {
        setChar(SEGMENTS_ALPHA[s - 97]);
    }
    // it cannot be displayed
    else {
        setChar(SEGMENTS_BLANK);
    }

    // special case for dots
    data[SEGMENTS[7]] = s == '.' ? HIGH : LOW;
}

// copy the segments into `data`
void setChar(const std::array<byte, 7> &s) {
    for (byte i = 0; i < 7; i++) {
        data[SEGMENTS[i]] = s[i];
    }
}
```

#### Display a string

We have all the necessary to display a complete string, we just have to loop on each character.

```c++
void loop() {
    // for each character
    for (byte i = 0; i < 20; i++) {
        // set the active grid
        data[GRID[i]] = 1;
        // set the segments
        setChar(i);

        // send the data
        for (int k = 19; k >= 0; k--) {
            digitalWrite(DRIVER_DIN, data[k]);
            clock();
        }
        load();

        // reset the active grid
        data[GRID[i]] = 0;
    }
}

void clock() {
    digitalWrite(DRIVER_CLK, HIGH);
    digitalWrite(DRIVER_CLK, LOW);
}

void load() {
    digitalWrite(DRIVER_LOAD, HIGH);
    digitalWrite(DRIVER_LOAD, LOW);
}
```

This `loop` method is called in the main Arduino loop, effectively blinking the segments very fast and show the text!

The complete implement supports other functions like blinking the whole display, force dots at specific positions, scroll messages longer than 12 characters and also display some simple special characters.

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
