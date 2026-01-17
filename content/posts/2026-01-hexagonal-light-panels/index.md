---
date: '2026-01-16'
title: 'Hexagonal light panels'
summary: 'Another take at the famous Nanoleaf/Govee hexa light panels.'
tags:
    - ESP32
    - WLED
cover:
    image: cover.jpg
---

## Introduction

A while ago a built some [moodlite light panels](https://www.printables.com/model/1417-moodlite-rgb-wall-mountable-lights), an open source model similar to Nanoleaf triangles. In fact I remixed the original model to made some improvements and iterated a few times the design, first with a custom controller, then by using WLED, adding magnets, adding small variants...

{{< image src="piwigo:2022/10/22/20221022180344-392b46a8" >}}

(you can find more photos of this project [here](https://photos.strangeplanet.fr/index.php?/category/191-moodlite_aout_2019_janvier_2020))

---

However the general quality is not so great and I wanted something more polished and beautiful. [This project video](https://www.youtube.com/watch?v=KsK9eldbPj0) by Smart Solutions for Home gave me the kick in the butt to build a new version.

Of course there are lot of models already available on Printables and other sites, but I don't like the general use of sharp angles and predetermined layouts. I want my light panels to have round corners, to be modular and extensible. The SS4H-LumiHex project is really nice but too expensive as it needs one MCU by tile and large custom PCBs.

So I went on a journey to design my one hexagonal light panels!

> All files for this project are available at the bottom of this page.


## The design

Each tile is a 18cm wide hexagon with 18mm rounded corners and is 25mm thick. The top is 3mm thick frosted PMMA, but can also be 3D printed.

{{< image src="images/render-tile.jpg" >}}

There is also a ring version.

{{< image src="images/render-tile-ring.jpg" >}}

As well as a tile acting as a small shelf for decorations or small plants (with three variants for the back).

{{< image src="images/render-tile-shelf.jpg" >}}

The controller box follows the same general shape. It houses a ESP8266 D1 Mini running WLED, has push button and a GX12-2 connector for power (also available for standard 5.5mm DC jack). In fact any MCU fitting inside the box can be used.

{{< image src="images/render-controller.jpg" >}}

### Hardware and mounting

A 60cm long LED strip is sticked in the inside edge (that's 36 LEDs for a standard 60 LED/meter strip). I used 12V WS2805 strips because of their additional warm white and cold white LEDs. Cheaper WS2812 or WS2815 can also be used.

{{< image src="images/render-strip.png" >}}

Each side has two recessed holes used to assemble the tiles together with 3D printed clamps. There are also small plugs used to close unused holes.
_Disclaimer:_ this system is 100% copied from [SS4H-LumiHex](https://smartsolutions4home.com/lumihex/).

{{< image src="images/render-clamps.png" >}} {{< image src="images/render-plugs.png" >}}

On the back, channels allow to route electric wires between connectors. A connector is composed of small PCB with a right angled female 2.54mm plug.

The connector on the bottom side of each tile is the one connected the strip (the tile cannot be re-oriented). This system allows tiles to be connected from any direction and be modular.

{{< image src="images/render-connector.png" >}}

The back of each tile has a hidden keyhole slot for wall mounting with its matching screw piece.

{{< image src="images/render-keyhole.png" >}}

The controller will be an ESP8266 running WLED, a single push button is used to quickly power on/off the fixture without using the app.

{{< image src="images/controller-schematic.png" >}}


## Bill of materials

### Tiles

For each tile (**full** or **ring**) the following parts are needed:

- 60cm long LED strip, 10mm wide or less
- one 3-pins 2.54mm L15 symmetric connectors ([on aliexpress](https://aliexpress.com/item/1005008747214913.html))
- one to six custom connector PCB (one for each connected side + bottom side)
- one to six 5-pins 2.54mm angled female socket ([on aliexpress](https://aliexpress.com/item/1005008210914833.htm))
- one 3mm frosted acrylic / 3D printed **cover** or **cover-ring**
- wires (recommended AWG20 for power)
- two to ten (two for each connected side) 3D printed **clamp**
- zero to ten (two for each un-connected side) 3D printed **plug**

The PCB must be 1.2mm thick, you can order 50 pieces for ~10€ on JLCPCB.

The acrylic panels are more expensive, I had them laser cut by [plaqueplastique.fr](https://plaqueplastique.fr/) (french company) for 6€ each.

{{< image src="photos/IMG_20260111_172016.jpg" title="Connectors pieces" >}}
{{< image src="photos/IMG_20260117_090814.jpg" title="ESP8266, buck converter and GX12 connector" >}}
{{< image src="photos/IMG_20260116_201613.jpg" title="Acrylic panels" >}}

### Shelves

- two or three M3 brass inserts
- two or three 12mm M3 screws

### Controller

- one 5 pins 2.54mm angled female socket
- one custom connector PCB
- one MCU of your choice
- one 3.3V or 5V buck converter if using 12V leds
- one 8mm push button ([aliexpress](https://aliexpress.com/item/1005007168217107.html))
- one 5.5mm DC jack or GX12-2 plug
- one power supply + power cables

Check the datasheet of your LED strips to know the required voltage and power. Be aware that WLED brightness limiter does not work well with WS2805 because the white leds are not taken into account ([open issue on GitHub](https://github.com/wled/WLED/issues/4132)).

I used a 12V/150W power supply from COXO ([aliexpress](https://aliexpress.com/item/1005005078402095.html)), with AWG18 cable and GX12-2 connector.


## Assembly

### 3D printing

Every part is designed to be printed flat without generated supports. There are is a small hoverhang with integrated supports on the model.

Recommended settings:

- white PLA
- 2 walls (3 for the **clamp**)
- 0.2mm layer height
- 10% gyroid infill
- no support
- Arachne wall generator

### Tiles

You can choose to entirely wire every 6 connectors of every tile, but this would be a waste of time and resources. I recommend you make a layout and only put connectors where needed, you will be able to come back later and add more if needed.

Here are the steps to assemble a tile:

1. Cut a 60cm length of LED strip and stick it to the inside of tile, make sure to begin on top of the square holes and to orient the strip clock-wise.

{{< image src="photos/IMG_20260113_220813.jpg" >}}

2. Solder the female sockets to the connector PCBs, making one for each connected side + the bottom side even if not connected. Solder by the top and trim the pins flush.

{{< image src="photos/IMG_20260113_220533.jpg" >}}

3. Place the connectors on the tile and wire all connectors together.

{{< image src="photos/IMG_20260113_220626.jpg" >}}
{{< image src="photos/IMG_20260115_194718.jpg" >}}

4. Wire the LED strip to the PCB. This is the most tedious step of the build! My design does no use the backup data line, if you have one you must solder BI and GND together.

{{< image src="photos/IMG_20260113_221624.jpg" >}}

(You will notice on this photo that wires are crossing each other, this was fixed in the final version of the PCB by inverting DI/DO and VCC/GND).

5. Test that the tile works correctly now before assembly!

### Shelves

For now just assemble the heated inserts, the shelf itself will be screwed later.

{{< image src="photos/IMG_20260111_171745.jpg" >}}

### Controller

The controller case requires the same connector assembly. Pay attention to solder the data wire to the `DO` pin of the connector.

The rest is only flying wires tucked in the case!

{{< image src="photos/IMG_20260109_230320.jpg" size=220 square=true >}}
{{< image src="photos/IMG_20260109_234914.jpg" size=220 square=true >}}
{{< image src="photos/IMG_20260111_172420.jpg" size=220 square=true >}}

If you use the same power supply as me you can use my simple protective case.

{{< image src="photos/IMG_20260111_152103.jpg" >}}

### Mounting

Once you have all your tiles, assemble them on a flat surface, front laying down:

1. Add a male-male pins header where needed. **WARNING**: Use 3 pins male-male headers, not 5! On the input side of a tile `DO` must not be connected to the previous tile.

{{< image src="images/data-direction.png" >}}

2. Use clamps to secure tiles together.

{{< image src="photos/IMG_20260116_125132.jpg" >}}

3. Use plugs to seal unused openings.

{{< image src="photos/IMG_20260116_125224.jpg" >}}

4. Now is a good time to ensure everything lights up correctly!

{{< image src="photos/IMG_20260116_171836.jpg" >}}
{{< image src="photos/IMG_20260116_182622.jpg" >}}

5. Temporarily insert a **screw-locator** on each tile you want screwed to the wall. You don't need a screw on every tile, 3-4 total should be enough.

{{< image src="photos/IMG_20260116_195245.jpg" >}}

6. Place the fixture on the wall, ensure it is leveled, then press firmly on each tile having a locator. This will mark where you need to drill the wall.

7. Fix the screw pieces with screws of your choices on the wall. I used raw drywall screws (the black ones).

{{< image src="photos/IMG_20260116_191459.jpg" >}}

8. Remove the screw locators and put on the shelves and the acrylic covers. You can use a bit of glue if needed.

9. Then mount the fixture on the wall, ensuring each screw is well seated in its keyhole.

{{< image src="piwigo:2026/01/17/20260117094520-24afbbd2" >}}


## Conclusion

This is what this project cost me as of January 2026 for 10 tiles and 3 shelves:

| item | price |
|---|---|
| ~1.5Kg White PLA | 17€ |
| 3m 22 AWG cable | 1.60€ |
| 6m WS2805 LED strip | 29€ |
| ESP8266 D1 Mini | 1.90€ |
| mini DC-DC buck converter | 0.70€ |
| GX12-2 connector | 0.75€ |
| x50 2.54mm female 5-pins header | 4€ |
| 2.54mm straight 40-pins header | 0.25€ |
| 8mm push button | 0.60€ |
| 12V 150W power supply | 4.35€ |
| x10 frosted acrylic panels | 77€ |
| x50 connector PCB | 12.55€ |
| **TOTAL** | **149.70€** |

So is it worth it? Nanoleaf Shapes Hexagons kit is 180€ for 9 panels. Govee Glide Hexa kit is 170€ for 10 panels. My version is a bit cheaper but considering the time invested – I estimate I spend ~15 hours designing the model and 20 minutes assembly by tile – and the lesser quality (mainly due to the diffuser) I would say: go to the retail product, unless you like making stuff yourself like I do!

The most expensive item is the acrylic panels. You can go a lot cheaper by 3D printing the front covers with translucent PLA (I didn't try) or if you have access to a laser cutter.

{{< image src="piwigo:2026/01/17/20260117094048-3d280166" >}}
{{< image src="piwigo:2026/01/17/20260117094309-9f37fc14" >}}
{{< image src="piwigo:2026/01/17/20260117094309-4bb997ae" >}}
{{< image src="piwigo:2026/01/17/20260117094308-1041f841" >}}

If you are interrested in creating the same light you can find 3D printing models, DXF and GERBER files on Printables: [Hexagonal light panels by mistic100](https://www.printables.com/model/1559320-hexagonal-light-panels).

And all the source files here: [StrangePlanet-HexaLightPanels_2026-01-16.zip (23.9 Mio)](https://www.strangeplanet.fr/files/Divers/StrangePlanet-HexaLightPanels_2026-01-16.zip)

![](https://licensebuttons.net/l/by-nc-sa/3.0/nl/88x31.png)
