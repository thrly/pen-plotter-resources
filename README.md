# Pen Plotting

notes from building and setting up a DIY pen plotter

![pen plotter](img/plotter.jpg)

## Building the plotter

### What?

I've used the X-Y plotter design (like the Axidraw/Next draw) [designed by Andrew Sleigh (v3)](https://andrewsleigh.github.io/plotter/). The design is simple, and the build was easy to work out. Andrew's design and guide is *excellent*. There isn't more I need to add there, go and build it!

### Notes on build

I 3D printed the parts (PLA seems fine) and opted for a 700mm v-slot X-axis, and a 500mm linear rails Y-axis. This gives me a working plotting area of about 420mm x 300 mm, which is frustratingly just a bit short to print A2 sized-plots. If I were doing this again, I would probably go for something more appropriate the A3. (Luckily, because DIY, I only have to buy a new v-slot and rails to upgrade).

I have noticed some slight 'drooping' on the Y-axis arm, so going beyond the current size might cause problems there. If I do need to scale up, I might look at an H-system design, as seen in the [ACRO](https://www.instructables.com/ACRO-Openbuilds-Pen-Plotter-Arduino-With-GRBL-and-/), Unatek, and more recently, the Bantam ArtFrame designs.

The BOM on Sleigh's post is good. In the UK, I purchased most parts from <https://ooznest.co.uk>. I would guess that the whole build cost me under £150 (2024)?

Don't underestimate the amount/type/colour bolts you need.

I *hate* doing Dupont crimps by hand, but I hate the idea of buying a tool more.

### arduino + shield + drivers

The plotter runs off an Arduino (Elegoo R3) with a CNC shield (protoneer clone). It's running GRBL, controlled with G-code, like a CNC machine. On the CNC shield, I opted for TMC2208 motor drivers with NEMA17 Stepper Motors (1.8° 77oz - probably overkill), and I'm really happy with the lower noise and performance so far. I [dialled the Vref](https://all3dp.com/2/vref-calculator-tmc2209-tmc2208-a4988/) down to about 0.8V on each driver (ymmv) and with the fan and heatsinks it seems happy. Before that, the drivers and motors got a bit warm during a plot.

For the CNC shield, note that you'll need to make a small modification in order to draw 12V power from the Arduino to power the motors, rather than use a second power supply. Sleigh mentions it in the guide, but don't overlook it.

### modifications (TODO: add .stl files for these)

Sleigh's design worked brilliantly for me (apart from the Arduino box fit/print time), but I've made a few small modifications to the 3D printer parts:

- **A new mounting bracket for the Arduino and CNC shield.** I found the original box didn't account for heat sinks, was fiddly to install and took a long time to print. I've opted for an open mount, with a bracket to hold a 40 mm fan pointed at the heat sinks which works well.
- **Remixed Sleigh's pen holder design for fountain pens.** I've split this into two parts, allowing the pen-holder to rotate to 45 degrees. This allows it to be used with fountain pens, like the Axi/Next Draw. It's not perfect, I'll probably keep tweaking this. If you just want normal vertically mounted pens, use the Sleigh design.
- **Foot risers.** This was to give more space for vertically mounted pens when I was printing on a seperate board. Now that I've mounted the plotter on a large MDF sheet, the risers are probably unneccessary. Also, with 45 degree fountain pens, the gap is actually a bit big... so I'll probably remove these soon.

![pen plotter attachment holding a fountain pen at 45 degrees](img/fountain_holder.jpg)
![3D printed bracket holding arduino and cnc shield with fan](img/arduino_bracket.jpg)

### limit switches

Not essential, but I've recently added (x2) limit switches to the 0 ends of my x and y axes so that I can home the plotter. This isn't essential and I'm not sure how much it helps. There's a bit of a confusion on the CNC shield, even though there's two jumpers (+ and -) for each axis limit switch... connecting both ends doesn't work. You need to wire the switches in series and just connect to one of the jumpers (and ground). The cabling started to get a bit messy with both ends' limit switches, so I've just opted for one switch per axis for now.

### grbl calibration settings for plotter

Once assesmbled, the plotter [needs calibrating](https://github.com/gnea/grbl/wiki/Grbl-v1.1-Configuration). Szymon Kaliski has some useful notes [here](https://szymonkaliski.com/writing/2023-10-02-building-a-diy-pen-plotter/#grbl). I found that [UGS](https://winder.github.io/ugs_website/) offers a helpful GUI wizard for calculating and setting things like travel resolution and limit/end-switches. You can also set these commands individually in the CNCJS console. This is obviously specific to every machine. As a starting point, my settings are here. (TOODO: add settings file)

### GRBL (Pen Servo flavour)

As outlined in the plotter design notes, you need to make a couple of modifications to the grbl library. I had to modify the pen-up/down positions in `spindle_control.c` before uploading to Arduino:

```
#define PEN_SERVO_DOWN     31      (maybe this should come up a little higher?)
#define PEN_SERVO_UP       22
```

In the GCode this equates to `Z0` = pen down, and `Z>0` = pen up

***

## Preparing to Plot - notes on my workflow

So the plotter is built, and everything is calibrated and tested. Pen and paper ready! How to plot?

1. Create your SVG in Processing/P5js/Inkscape.
2. Optimise SVG and convert to GCode with `vpype` and `vpype-gcode`.
3. Preview plot with vpype's `show` command instead of final output. E.g.: `vpype read input.svg linemerge linesort linesimplify show` to check it looks good
4. Plot! Send GCode to modified GRBL on Arduino/CNC shield, controlling the plotter (this has to be sent 'live', so make sure your computer doesn't sleep). I'm using [cncjs](https://cnc.js.org/) for this control.

#### exporting SVGs from p5js

P5js doesn't support SVG export natively. Until recently we had to use the p5.js-svg library which has been out of date for a while. Golan Levin has now come to the rescue with [P5.PlotSVG](https://github.com/golanlevin/p5.plotSvg) - SVG rendering, specifically for pen plotters!

### command to convert .SVG into ready-to-print GCode

`vpype -c config.toml read --no-crop input.svg layout a4 pagerotate occult -i linemerge linesort linesimplify reloop reloop gwrite --profile fast-plot output.gcode`

This loads the custom config .toml file, in your working directory. This is where I've set my plotter profiles which formats the gcode. Here is my default 'fast-plot' profile:

#### vpype-gcode custom profiles

```
[gwrite.fast-plot]
unit = "mm"
document_start = """
G90       ; absolute coordinates
G21       ; mm units
G17       ; xy plane
G00 Z1    ; pen up"""

layer_start = "(Start Layer)\n"
line_start = "(Start Layer)\n"

segment_first = """(Start Segment)
G00 Z1 (pen up)
G00 X{x:.4f} Y{y:.4f} (travel)
G00 Z0(pen down)\n"""

segment = """G00 X{x:.4f} Y{y:.4f}\n"""

line_end = """(Line End)
G00 Z1 (pen up)"""

document_end = """(Document End)
G00 Z1 (pen up)
G00 X0.0 Y0.0
M2"""

invert_y = true

info= "FAST PLOT. All G00 movements. Gcode ready to print. File created succesfully."
```

## splitting a multi-coloured SVG into to multi-layer gcode

Sometimes you'll want to plot in multiple pens, colours, etc. To do that, you can split an SVG into multiple layers, each layer getting its own gcode to plot (be careful to align your pen nib for each layer...)
In vpype, read the svg, splitting layers by stroke (`read -a stroke`), then delete all layers, keeping the one you want (`ldelete --keep 1`)
For example:
 First, inspect your layers to see which is which (take note of the layer number):

 `vpype read --no-crop -a stroke circle-packed.svg show`

 Then run the command to render your gcode for the desired layer:

 `vpype -c thrly-config.toml read --no-crop -a stroke circle-packed.svg layout -m 3cm 282x210mm pagerotate ldelete --keep 1 linemerge reloop linesort gwrite -p fast-plot circle-pack-layer-1.gcode`

 Ensure that you delete the layers *after* setting your layout/scale/position, otherwise the layers will be out of alignment with each other.
 Then repeat for each layer.

\[EDIT, Nov '24: that said, I think I've had some problems recently with alignment between layers, so maybe something is off here?]

## Summary of resources

- the **plotter** -  <https://github.com/andrewsleigh/plotter/tree/master>
- **grbl** pen servo, running on the arduin + cnc shield - download and install instead of vanilla GRBL (install in the same way) <https://github.com/bdring/Grbl_Pen_Servo/tree/master>
- **Processing** or **p5js** for designing generative stuff.
- Inkscape -- I'm not using it for design/layout, but its occasionally useful for debugging a weird SVG file.
- P5.PlotSVG - SVG rendering, specifically for pen plotters - <https://github.com/golanlevin/p5.plotSvg>
- ~P5.js-SVG library, because P5js doesn't actually support SVGs (only working with P5js v. 1.6.0)- <https://github.com/zenozeng/p5.js-svg> (depreciated in my workflow)~
- **vpype** for optimising svg for plotting - <https://github.com/abey79/vpype>
- **vpype-gcode** (plugin) for converting svg to gcode - <https://github.com/plottertools/vpype-gcode>
- vpype **occult** (plugin) for hidden line removal
- **cncjs** - <https://cnc.js.org/>

## Future plans

- Redesign pen holder to make servo more easily replaceable
- Switch v-slot aluminium and linear rods to fit A2 plots (or possibly move over to an H-frame design.)

## TODO

- add images
- Add stl fils for my 3d print mods

![a cat interfering with plotter setup](img/setup_cat.jpg)