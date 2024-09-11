# thrly_custom_plotter
 notes from building and setting up a DIY pen plotter

# Tools I'm using

- vpype - https://github.com/abey79/vpype
- vpype-gcode (plugin) - https://github.com/plottertools/vpype-gcode
- vpype occult (plugin)
- Universal Gcode Sender - https://universalgcodesender.com/
- (or) cncjs
- grbl pen servo - download and install instead of vanilla GRBL (install in the same way) https://github.com/bdring/Grbl_Pen_Servo/tree/master
- the plotter -  https://github.com/andrewsleigh/plotter/tree/master
- axidraw CLI / inkscape plugins - for removing hidden layers with `--hiding` (works better than `vpype occult -i` - https://axidraw.com/doc/cli_api/#installation [EDIT: this didn't really work as well as hoped on the Truchet patterns...]

# grbl settings for plotter
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
$32=0 (Laser-mode enable, boolean)
$100=53.273 (X-axis travel resolution, step/mm)
$101=80.355 (Y-axis travel resolution, step/mm)
$102=250.000 (Z-axis travel resolution, step/mm)
$110=8000.000 (X-axis maximum rate, mm/min)
$111=8000.000 (Y-axis maximum rate, mm/min)
$112=1000.000 (Z-axis maximum rate, mm/min)
$120=200.000 (X-axis acceleration, mm/sec^2)
$121=200.000 (Y-axis acceleration, mm/sec^2)
$122=100.000 (Z-axis acceleration, mm/sec^2)
$130=200.000 (X-axis maximum travel, millimeters)
$131=200.000 (Y-axis maximum travel, millimeters)
$132=200.000 (Z-axis maximum travel, millimeters)
```
# Notes

## workflow
1. Create .SVG in Processing/P5js/Inkscape
2. Optimise SVG and convert to GCode with `vpype` and `vpype-gcode`. For SVGs with a lot of hidden layers, I've found the Axidraw CLI `--hiding` tool works better than vpype's `occult` plugin.
   If using AxiCLI use: `axicli file.svg -h -o outputfile.svg` to remove hidden lines.
   Preview plot with vpype's `show` command instead of final output. E.g.: `vpype read input.svg occult -i linemerge linesort linesimplify show`
4. Send GCode to modified GRBL on Arduino/CNC shield, controlling the plotter (this has to be done 'live').

## command to convert .SVG into ready-to-print GCode:

`vpype --config thrly-config.toml read --no-crop input.svg layout a4 pagerotate occult -i linemerge linesort linesimplify reloop gwrite --profile plotter output.gcode`

loads the custom config toml file, located in whichever directory you need it. The contents of that file is:

```
########################################################################################################################
# Custom Profile
########################################################################################################################
[gwrite.plotter]
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

info= "Profile settings stored in thrly-config.toml\nInverted across the y-axis. Gcode ready to print. File created succesfully."
```

## GRBL (Pen Servo)
I had to modify the pen-up/down positions in `spindle_control.c` before uploading to Arduino:
```
#define PEN_SERVO_DOWN     31      (maybe this should come up a little higher?)
#define PEN_SERVO_UP       25
```

In the GCode, Z0 = pen down; Z>0 = pen up

Once the plotter is setup and running, it needs calibrating via GRBL. See: https://szymonkaliski.com/writing/2023-10-02-building-a-diy-pen-plotter/ and https://github.com/gnea/grbl/wiki/Grbl-v1.1-Configuration 

## To Do
- Machine calibration details (from UGS?)
- details of CNC shield assembly + 12V mod
- link to Andrew Sleigh's site / 3d print files
- notes on build
- notes on software setup

## Future plans
- Redesign pen holder to make servo more easily replaceable
- Redesign pen holder to make detachable and easily mountable at 45 degrees (see Axidraw design)
- Taller feet with spacer for board/paper alignment
- Flip for left-right motion, better 0,0 home (or use invert commands in vpype?)
- End stop limit-switches for homing? (ordered, spet 2024)
