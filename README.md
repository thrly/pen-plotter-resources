# thrly_custom_plotter
 setting up DIY pen plotter

# Tools I'm using

- vpype - https://github.com/abey79/vpype
- vpype-gcode (plugin) - https://github.com/plottertools/vpype-gcode
- Universal Gcode Sender - https://universalgcodesender.com/
- (or) cncjs
- grbl pen servo - download and install instead of vanilla GRBL (install in the same way) https://github.com/bdring/Grbl_Pen_Servo/tree/master

## command to convert .SVG into ready-to-print GCode:
`vpype --config thrly-config.toml read in_file.svg linemerge linesort gwrite --profile thrly output_file.gcode`
This command loads the custom config toml file, located in whichever directory you need it. The contents of that file is:

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
I had to modify the pen-up/down positions in `spindle_control.c`:
```
#define PEN_SERVO_DOWN     31      
#define PEN_SERVO_UP       25
```

In the GCode, Z0 = pen down; Z>0 = pen up