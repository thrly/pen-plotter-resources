# Pen Plotting

notes from building and setting up a DIY pen plotter

## Building the plotter

I've used the X-Y plotter design (like the Axidraw/Next draw) [designed by Andrew Sleigh (v3)](https://andrewsleigh.github.io/plotter/). I 3D printed the parts (PLA seems fine) and opted for a 700mm v-slot X-axis, and a 500mm linear rails Y-axis. This gives me a working plotting area of about 420mm x 300 mm, which is frustratingly a bit short to print A2 sized-plots. (Luckily, because DIY, I only have to buy a new v-slot and rails to upgrade. I have noticed some 'drooping' on the Y-axis, so going beyond the current size might cause problems there. If I do need to scale up, I might look at an H-system...)(in the UK, I purchased most parts from <https://ooznest.co.uk>). I would guess that the whole build cost me under £150 (2024)?

### arduino + shield + drivers

The plotter runs off an Arduino (Elegoo R3) with a CNC shield (protoneer clone). I opted for TMC2208 motor drivers with NEMA17 Stepper Motors (1.8° 77oz - probably overkill), and I'm really happy with the lower noise and performance so far. I [dialled the Vref](https://all3dp.com/2/vref-calculator-tmc2209-tmc2208-a4988/) down to about 0.8V on each driver and with the fan and heatsinks it seems happy.

### modifications

The only modifications to Sleigh's design were printing taller foot extenders (with a 90 degree jig for paper alignment, which doesn't help) and a redesigned arduino bracket. I found the original box didn't account for heat sinks, was fiddly to install and took a long time to print. I've opted for an open mount, with a bracket to hold a 40 mm fan pointed at the heat sinks... (TODO: add .stl files for these)

### limit switches

I've recently added (x2) limit switches to the 0 ends of my x and y axes so that I can home the plotter. This isn't essential and I'm not sure how much it helps. There's a bit of a confusion on the CNC shield, even though there's two jumpers (+ and -) for each axis limit switch... connecting both ends doesn't work. You need to wire the switches in series and just connect to one of the jumpers (and ground). The cabling started to get a bit messy with both ends' limit switches, so I've just opted for one switch per axis for now.

### grbl calibration settings for plotter

Once assesmbled, the plotter needs callibrating. I found that [UGS](https://winder.github.io/ugs_website/) offers a helpful GUI wizard for calculating and setting things like travel resolution and limit/end-switches. This is obviously specific to every machine. My settings are currently as follows:

```$0=10 (Step pulse time, microseconds)
$1=25 (Step idle delay, milliseconds)
$2=0 (Step pulse invert, mask)
$3=2 (Step direction invert, mask)
$4=0 (Invert step enable pin, boolean)
$5=0 (Invert limit pins, boolean)
$6=0 (Invert probe pin, boolean)
$10=1 (Status report options, mask)
$11=0.010 (Junction deviation, millimeters)
$12=0.002 (Arc tolerance, millimeters)
$13=0 (Report in inches, boolean)
$20=0 (Soft limits enable, boolean)
$21=0 (Hard limits enable, boolean)
$22=0 (Homing cycle enable, boolean)
$23=0 (Homing direction invert, mask)
$24=25.000 (Homing locate feed rate, mm/min)
$25=500.000 (Homing search seek rate, mm/min)
$26=250 (Homing switch debounce delay, milliseconds)
$27=1.000 (Homing switch pull-off distance, millimeters)
$30=1000 (Maximum spindle speed, RPM)
$31=0 (Minimum spindle speed, RPM)
$32=0 (Laser-mode enable, boolean)```
$100=53.273 (X-axis travel resolution, step/mm) ; calibration for X axes
$101=80.355 (Y-axis travel resolution, step/mm) ; calibration for Y axes
$102=250.000 (Z-axis travel resolution, step/mm)
$110=8000.000 (X-axis maximum rate, mm/min) ; set speed motor (this could maybe go up, grbl docs say find max and set to 80-90%)
$111=8000.000 (Y-axis maximum rate, mm/min) ; set speed motor (this could maybe go up, grbl docs say find max and set to 80-90%)
$112=1000.000 (Z-axis maximum rate, mm/min)
$120=200.000 (X-axis acceleration, mm/sec^2) ; this is like the rate, but specifically acceleration. again this could maybe go up, but I'm being cautious for jerking the pen
$121=200.000 (Y-axis acceleration, mm/sec^2) ; this is like the rate, but specifically acceleration. again this could maybe go up, but I'm being cautious for jerking the pen
$122=100.000 (Z-axis acceleration, mm/sec^2)
$130=200.000 (X-axis maximum travel, millimeters)
$131=200.000 (Y-axis maximum travel, millimeters)
$132=200.000 (Z-axis maximum travel, millimeters)
```

## Preparing to Plot - Workflow

1. Create .SVG in Processing/P5js/Inkscape
2. Optimise SVG and convert to GCode with `vpype` and `vpype-gcode`. For SVGs with a lot of hidden layers, I've found the Axidraw CLI `--hiding` tool works better than vpype's `occult` plugin.
   Preview plot with vpype's `show` command instead of final output. E.g.: `vpype read input.svg occult -i linemerge linesort linesimplify show` to check it looks good
4. Send GCode to modified GRBL on Arduino/CNC shield, controlling the plotter (this has to be done 'live'). I'm using [cncjs](https://cnc.js.org/) most of the time, but [UGS](https://winder.github.io/ugs_website/) seems okay too.

### command to convert .SVG into ready-to-print GCode

`vpype --config thrly-config.toml read --no-crop input.svg layout a4 pagerotate occult -i linemerge linesort linesimplify reloop reloop gwrite --profile plotter output.gcode`

loads the custom config toml file, located in whichever directory you need it. The contents of that file is:

## vpype-gcode custom profiles
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
see .toml file for other profiles.

## splitting a multi-coloured SVG into to multi-layer gcode
Sometimes you'll want to plot in multiple pens, colours, etc. To do that, you can split an SVG into multiple layers, each layer getting its own gcode to plot (be careful to align your pen nib for each layer...)
In vpype, read the svg, splitting layers by stroke (`read -a stroke`), then delete all layers, keeping the one you want (`ldelete --keep 1`)
For example:
 First, inspect your layers to see which is which (take note of the layer number):
 
 `vpype read --no-crop -a stroke circle-packed.svg show`
 
 Then run the command to render your gcode for the desired layer:
 
 `vpype -c thrly-config.toml read --no-crop -a stroke circle-packed.svg layout -m 3cm 282x210mm pagerotate ldelete --keep 1 linemerge reloop linesort gwrite -p fast-plot circle-pack-layer-1.gcode`
 
 Ensure that you delete the layers _after_ setting your layout/scale/position, otherwise the layers will be out of alignment with each other.
 Then repeat for each layer.

## GRBL (Pen Servo)

I had to modify the pen-up/down positions in `spindle_control.c` before uploading to Arduino:

```
#define PEN_SERVO_DOWN     31      (maybe this should come up a little higher?)
#define PEN_SERVO_UP       22
```

In the GCode, Z0 = pen down; Z>0 = pen up

Once the plotter is setup and running, it needs calibrating via GRBL. See: <https://szymonkaliski.com/writing/2023-10-02-building-a-diy-pen-plotter/> and <https://github.com/gnea/grbl/wiki/Grbl-v1.1-Configuration>

## End stops
I added end-stop limit switches to the X and Y axes, but I'm not sure how useful they are in reality. Sure it will stop a collision, but adds the hassle of unlocking the arm each time I turn the machine on. I'm rarely working with files that aren't well within the limits, and if I check my scaling and measurements, collisions are unlikely.

## Tools I'm using

- the plotter -  <https://github.com/andrewsleigh/plotter/tree/master>
- grbl pen servo - download and install instead of vanilla GRBL (install in the same way) <https://github.com/bdring/Grbl_Pen_Servo/tree/master>
- Processing or P5JS for designing stuff
- vpype - <https://github.com/abey79/vpype>
- vpype-gcode (plugin) - <https://github.com/plottertools/vpype-gcode>
- vpype occult (plugin) for hidden line removal
- cncjs - <https://cnc.js.org/>
- also some luck with - Universal Gcode Sender - <https://universalgcodesender.com/>


## Details to add

- add images
- Machine calibration details (from UGS?)
- details of CNC shield assembly + 12V mod
- link to Andrew Sleigh's site / 3d print files
- notes on build
- notes on software setup

## Future plans

- Redesign pen holder to make servo more easily replaceable
- Redesign pen holder to make detachable and easily mountable at 45 degrees for fountain pens (see Axidraw design)
- Add stl fils for my custom board mount and feet.
