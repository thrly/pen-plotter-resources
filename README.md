# thrly_custom_plotter
 notes from building and setting up a DIY pen plotter

# Tools I'm using

- vpype - https://github.com/abey79/vpype
- vpype-gcode (plugin) - https://github.com/plottertools/vpype-gcode
- Universal Gcode Sender - https://universalgcodesender.com/
- (or) cncjs
- grbl pen servo - download and install instead of vanilla GRBL (install in the same way) https://github.com/bdring/Grbl_Pen_Servo/tree/master
- the plotter -  https://github.com/andrewsleigh/plotter/tree/master

# Notes

## workflow
1. Create .SVG in Processing/P5js/Inkscape
2. Optimise SVG and convert to GCode with `vpype` and `vpype-gcode`
3. Send GCode to modified GRBL on Arduino/CNC shield, controlling the plotter (this has to be done 'live').

## command to convert .SVG into ready-to-print GCode:

`vpype --config thrly-config.toml read input.svg linemerge linesort gwrite --profile thrly output.gcode`

loads the custom config toml file, located in whichever directory you need it. The contents of that file is:

```
[gwrite.thrly]
unit = "mm"
document_start = "G21\nG17\nG90\n"
layer_start = "(Start Layer)\n"
line_start = "(Start Block)\n"
segment_first = """G00 Z1 (pen up)
G00 X{x:.4f} Y{y:.4f} (travel)
G1 Z0 F6000\n (pen down)
"""
segment = """G01 X{x:.4f} Y{y:.4f} F10000\n"""
line_end = """G00 Z 5.0000
M5 S0
G4 P0.5\n"""
document_end = """M5
G00 X0.0000 Y0.0000
M2"""
invert_y = true
info= "Inverted across the y-axis. Gcode ready to print. File created succesfully."
```

## GRBL (Pen Servo)
I had to modify the pen-up/down positions in `spindle_control.c` before uploading to Arduino:
```
#define PEN_SERVO_DOWN     31      
#define PEN_SERVO_UP       25
```

In the GCode, Z0 = pen down; Z>0 = pen up

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
- Flip for left-right motion, better 0,0 home
- End stop limit-switches for homing?
