    # Aerotech G-code Support in Slic3r

This document describes the Aerotech G-code flavor implementation in Slic3r, which is specifically designed for high-precision Aerotech motion controllers with integrated laser control.

## Key Features

- 5nm resolution support (6 decimal places precision)
- Proper laser shutter control (M64/M65)
- Separate movement commands for better motion control
- Units per second feed rate
- Path blending and circular interpolation support
- Aerotech-style comments using //

## Program Structure

### Initialization Sequence
Every Aerotech G-code program starts with:
```gcode
M65     // Close shutter
G71     // Metric units (millimeters)
G76     // Time in seconds
G94     // Units per second mode
G90     // Absolute positioning
G109    // Disable velocity blending
G69     // Path blending with S curve
G16 X Y Z // Configure circular interpolation axes
G17     // Select XY plane for circular interpolation
```

### Program Termination
Programs end with:
```gcode
M65     // Laser off
G90     // Ensure absolute mode
G1 Z15.000000 F1000.000000 // Safe Z height
M2      // Program end
```

## G-code Commands

### Movement Commands
Linear movements use G1 with separate commands for each axis for better control:
```gcode
G1 X0.011000     // X movement with 5nm resolution
G1 Y0.381000     // Y movement
G1 Z-17.480000   // Z movement
F11.000000       // Feed rate in units per second
```

### Circular Interpolation
Circular movements use G2 (clockwise) or G3 (counterclockwise):
```gcode
G3 X0.024000 Y0.376000 I0.024000 J0.060000 // Arc with center coordinates
F11.000000
```

### Laser Control
Laser shutter control is managed using:
```gcode
M64     // Open shutter (laser on)
M65     // Close shutter (laser off)
```

The shutter is automatically controlled during movements:
- Closed before layer changes or Z movements
- Opened after reaching the target position
- Closed during non-printing moves

## Parameter Mapping

### Feed Rate (F)
- Uses units per second instead of units per minute
- Specified with 6 decimal precision
- Example: `F11.000000`

### Coordinates (X, Y, Z)
- All coordinates use 6 decimal places for 5nm resolution
- Example: `X0.011000 Y0.381000 Z-17.480000`

### Movement Differences from Mach3
In Mach3, the A axis is used for extrusion control and comments use semicolons:
```gcode
// Mach3 format:
G1 X0.011 Y0.381 A0.00001 // infill with extrusion

// Aerotech format (no extrusion, laser controlled by M64/M65):
G1 X0.011000 Y0.381000    // infill movement
```

### Temperature and Fan Control
- Uses 'P' parameter instead of 'S'
- Example: `M104 P200 // Set temperature to 200°C`

## Example G-code

Here's a complete example of a layer change and infill:
```gcode
M65                     // Close shutter before layer movement
G1 Z-17.480000         // Layer change
G1 X0.160000 Y0.418000 // Travel to first layer point
M64                    // Open shutter after layer movement
F11.000000             // Set feed rate
G1 X-0.057000 Y0.442000 // Infill movement
G1 X-0.043000 Y0.445000 // Continue infill
M65                    // Close shutter
```

## Implementation Details

The Aerotech G-code flavor is implemented in the following files:
- `PrintConfig.hpp`: Defines the `gcfAerotech` flavor enum
- `GCodeWriter.cpp`: Implements the G-code generation logic

Key implementation features:
1. Higher precision output using `AEROTECH_NUM` macro (6 decimal places)
2. Automatic shutter control during movements
3. Separated axis movements for better control
4. Path blending and circular interpolation support
5. Proper feed rate handling in units per second
6. No extrusion axis (unlike Mach3's A axis)
7. Comments using // instead of Mach3's ;

## Usage

To use Aerotech G-code flavor in Slic3r:
1. Select "aerotech" as the G-code flavor in your printer settings
2. Configure your printer's dimensions and other parameters as usual
3. The generated G-code will automatically use Aerotech-specific commands and formatting

## Notes

- All movements are done in absolute coordinates (G90)
- Circular interpolation is configured for XY plane (G17)
- Path blending uses S-curve for smooth acceleration/deceleration
- The shutter is automatically controlled for optimal laser operation
- Feed rates are in units per second as per Aerotech specifications
- No extrusion axis is used; laser control is handled by M64/M65 commands
- Comments use // instead of ; for better compatibility with Aerotech controllers

## Mach3-LinuxCNC G-codes

| G-Code    | Function                                          |
|-----------|---------------------------------------------------|
| G0        | Rapid positioning                                 |
| G1        | Linear interpolation                              |
| G2        | Clockwise circular/helical interpolation          |
| G3        | Counterclockwise circular/helical interpolation   |
| G4        | Dwell                                             |
| G10       | Coordinate system origin setting                  |
| G12       | Clockwise circular pocket                         |
| G13       | Counterclockwise circular pocket                  |
| G15/G16   | Polar coordinate moves in G0 and G1               |
| G17       | XY plane select                                   |
| G18       | XZ plane select                                   |
| G19       | YZ plane select                                   |
| G20/G21   | Inch/millimeter unit selection                    |
| G28       | Return home                                       |
| G28.1     | Reference axes                                    |
| G30       | Return home (secondary position)                  |
| G31       | Straight probe                                    |
| G40       | Cancel cutter radius compensation                 |
| G41/G42   | Start cutter radius compensation left/right       |
| G43       | Apply tool length offset (plus)                   |
| G49       | Cancel tool length offset                         |
| G50       | Reset all scale factors to 1.0                    |
| G51       | Set axis data input scale factors                 |
| G52       | Temporary coordinate system offsets               |
| G53       | Move in absolute machine coordinate system        |
| G54       | Use fixture offset 1                              |
| G55       | Use fixture offset 2                              |
| G56       | Use fixture offset 3                              |
| G57       | Use fixture offset 4                              |
| G58       | Use fixture offset 5                              |
| G59       | Use fixture offset 6 / use general fixture number |
| G61/G64   | Exact stop/constant velocity mode                 |
| G68/G69   | Rotate program coordinate system                  |
| G70/G71   | Inch/millimeter unit selection                    |
| G73       | Canned cycle – peck drilling                      |
| G80       | Cancel motion mode (including canned cycles)      |
| G81       | Canned cycle – drilling                           |
| G82       | Canned cycle – drilling with dwell                |
| G83       | Canned cycle – peck drilling                      |
| G84       | Canned cycle – right-hand rigid tapping           |
| G85-G89   | Canned cycle – boring                             |
| G90       | Absolute distance mode                            |
| G91       | Incremental distance mode                         |
| G92       | Offset coordinates and set parameters             |
| G92.x     | Cancel G92 offsets                                |
| G93       | Inverse time feed mode                            |
| G94       | Feed per minute mode                              |
| G95       | Feed per revolution mode                          |
| G98       | Initial level return after canned cycles          |
| G99       | R-point level return after canned cycles          |

### M-Codes Mach3

| M-Code    | Function                          |
|-----------|-----------------------------------|
| M0        | Program stop                      |
| M1        | Optional program stop             |
| M3        | Spindle on (clockwise rotation)   |
| M4        | Spindle on (counterclockwise)     |
| M5        | Spindle stop                      |
| M6        | Tool change                       |
| M7        | Mist coolant on                   |
| M8        | Flood coolant on                  |
| M9        | All coolant off                   |
| M30       | Program end and rewind            |
| M47       | Repeat program from first line    |
| M48       | Enable speed and feed override    |
| M98       | Call subroutine                   |
| M99       | Return from subroutine/repeat     |

## Aerotech G-code Commands

| G-Code    | Function                          |
|-----------|-----------------------------------|
| G0        | Executes a rapid point-to-point positioning move on the specified axes. |
| G1        | Executes a coordinated linear move on the specified axes. |
| G2        | Executes a circular arc move in the clockwise direction on the specified axes in the first coordinate plane. |
| G3        | Executes a circular arc move in the counterclockwise direction on the specified axes in the first coordinate plane. |
| G4        | Suspends the execution of the task for the duration that you specify, in seconds. |
| G8        | Causes the G-code motion that you specify on the same program line to immediately accelerate to the programmed feedrate. |
| G9        | Causes the G-code motion command that you specify on the same program line to decelerate to zero velocity when the axes get to their target position. |
| G12       | Executes a circular arc move in the clockwise direction on the specified axes in the second coordinate plane. |
| G13       | Executes a circular arc move in the counterclockwise direction on the specified axes in the second coordinate plane. |
| G16       | Configures the set of axes that are used for the first coordinate plane for G2 and G3 commands. |
| G17       | Configures the task to use the first set of axes of the first coordinate system for G2 and G3 commands. |
| G18       | Configures the task to use the second set of axes of the first coordinate system for G2 and G3 commands. |
| G19       | Configures the task to use the third set of axes of the first coordinate system for G2 and G3 commands. |
| G26       | Configures the set of axes that are used for the second coordinate plane for G12 and G13 commands. |
| G27       | Configures the task to use the first set of axes of the second coordinate system for G12 and G13 commands. |
| G28       | Configures the task to use the second set of axes of the second coordinate system for G12 and G13 commands. |
| G29       | Configures the task to use the third set of axes of the second coordinate system for G12 and G13 commands. |
| G40       | Deactivates cutter radius compensation mode on the task. |
| G41       | Activates cutter radius compensation left mode on the task. |
| G42       | Activates cutter radius compensation right mode on the task. |
| G43       | Specifies the radius that the task uses in cutter radius compensation mode. |
| G44       | Specifies the axes that comprise the X-Y or X-Y-Z plane when a cutter compensation mode is active on the task. |
| G53       | Disables work offsets. |
| G54       | Enables work offset 1. |
| G55       | Enables work offset 2. |
| G56       | Enables work offset 3. |
| G57       | Enables work offset 4. |
| G58       | Enables work offset 5. |
| G59       | Enables work offset 6. |
| G60       | Configures the task to use the acceleration duration that you specify for coordinated motion while operating in time-based ramping mode. |
| G61       | Configures the task to use the deceleration duration that you specify for coordinated motion while operating in time-base ramping mode. |
| G63       | Configures the task to operate with the sinusoidal acceleration and deceleration ramping type for coordinated motion. |
| G64       | Configures the task to operate with the linear acceleration and deceleration ramping type for coordinated motion. |
| G65       | Configures the task to use the acceleration rate that you specify for coordinated motion while operating in rate-based ramping mode. |
| G66       | Configures the rate that the task uses to perform coordinated motion to decelerate dominant axes. |
| G67       | Configures the task to use time-based ramping to accelerate and decelerate axes during coordinated motion. |
| G68       | Configures the task to use rate-based ramping to accelerate and decelerate axes during coordinated motion. |
| G69       | Configures the task to operate with the S-curve acceleration and deceleration ramping type for coordinated motion. |
| G70       | Configures the task to treat distances and feedrates that you specify in G-code commands as being English units. |
| G71       | Configures the task to treat distances and feedrates that you specify in G-code commands as being metric units. |
| G75       | Configures the task to treat feedrates that you specify in G-code commands as being distance units per minute. |
| G76       | Configures the task to treat feedrates that you specify in G-code commands as being distance units per second. |
| G82       | Clears the program position offset on the specified axis. The program position will be restored to the current axis position. |
| G90       | Configures the task to operate in the absolute target mode. |
| G91       | Configures the task to operate in the incremental target mode. |
| G92       | Sets the program position of the specified axes. All moves that specify an absolute target-position will be relative to the new program position. |
| G93       | Configures the task for inverse time feedrate mode. Values specified to the F command are interpreted as the inverse amount of time that the controller should take to complete the distance specified in the move. |
| G94       | Configures the task for normal feedrate mode. Values specified to the F command are interpreted as distance units per unit of time. |
| G95       | Configures the task to interpret feedrates that you specify with the F command or the E command as units per spindle revolution. |
| G96       | Configures the task to interpret the S command as the tangential speed, or the speed on the surface, of the spindle axis. You must specify surface speed and a radial axis to this command. |
| G97       | Configures the task to interpret the S command in velocity units. |
| G98       | Enables inverse dominance mode on the task. |
| G99       | Disables inverse dominance mode on the task. |
| G100      | Disables spindle shutdown mode on the task. |
| G101      | Enables spindle shutdown mode on the task. |
| G108      | Enables velocity blending mode on the task. |
| G109      | Disables velocity blending mode on the task. |
| G114      | Enables optional pause mode on the task. |
| G115      | Disables optional pause mode on the task. |
| G118      | Configures the center point coordinates that you specify to a coordinated arc motion command to be in absolute user units. |
| G120      | Configures the task to disable the effects of manual feedrate override (MFO) and the feedhold state during asynchronous motion. |
| G121      | Configures the task to enable the effects of manual feedrate override (MFO) and the feedhold state during asynchronous motion. |
| G143      | Activates cutter offset compensation positive mode on the task. |
| G144      | Activates cutter offset compensation negative mode on the task. |
| G149      | Deactivates cutter offset compensation mode on the task. |
| G150      | Disables part scaling mode on the task. |
| G151      | Enables part scaling mode on the task. |
| G153      | Suppresses work offsets for the G-code motion that you specify on the same program line. |
| G165      | Configures the rate that the task uses to perform coordinated motion to accelerate dependent axes. |
| G166      | Configures the rate that the task uses to perform coordinated motion to decelerate dependent axes. |
| G359      | Sets the wait mode to wait the minimum amount of time between motion blocks. |
| G360      | Sets the wait mode to wait for all motion to be done between motion blocks. |
| G361      | Sets the wait mode to wait for all motion to be done and for all axes to be in position between motion blocks. |

### M-Codes Aerotech

| M-Code    | Function                          |
|-----------|-----------------------------------|
| M0        | Pauses the current task.          |
| M1        | Pauses the current task if the optional pause mode is active on the task, or does nothing if optional pause mode is not active. |
| M2        | Stops the execution of the program on the current task. |
| M3        | Starts motion in the clockwise direction on the spindle axis that is configured for the task. The command is complete after the spindle accelerates to the commanded speed. |
| M4        | Starts motion in the counterclockwise direction on the spindle axis that is configured for the task. The command is complete after the spindle accelerates to the commanded speed. |
| M5        | Causes the spindle axis that is configured for the task to decelerate to zero speed. |
| M30       | Resets the execution of the AeroScript to the first executable line. |
| M47       | Resets the execution of the AeroScript to the first executable line. Then it causes the controller to start execution of the program. |
| M48       | Disables the use of manual feedrate override (MFO) controls. |
| M49       | Enables the use of manual feedrate override (MFO) controls. |
| M50       | Disables the use of manual spindle override (MSO) controls. |
| M51       | Enables the use of manual spindle override (MSO) controls. |
| M64       | Laser on                          |
| M65       | Laser off                         |
| M103      | Starts motion on the spindle axis that is configured for the task in the clockwise direction. The command does not wait for the spindle to accelerate to the commanded speed. |
| M104      | Starts motion on the spindle axis that is configured for the task in the counterclockwise direction. The command does not wait for the spindle to accelerate to the commanded speed. |

## G-Codes Comparison: Mach3 Manual vs GCodeWriter Implementation

| G-Code    | Mach3 Manual Function                | GCodeWriter Implementation          |
|-----------|--------------------------------------|-------------------------------------|
| G0        | Rapid positioning                    | Implemented in travel_to_xy(), travel_to_xyz() |
| G1        | Linear interpolation                 | Implemented in extrude_to_xy(), extrude_to_xyz() |
| G2        | CW circular interpolation            | Implemented in travel_to_xy_G2G3IJ(), extrude_to_xy_G2G3IJ() |
| G3        | CCW circular interpolation           | Implemented in travel_to_xy_G2G3IJ(), extrude_to_xy_G2G3IJ() |
| G21       | Programming in millimeters           | Implemented in preamble() |
| G82       | Absolute distance mode for extrusion | Implemented in preamble() as M82 |
| G83       | Relative distance mode for extrusion | Implemented in preamble() as M83 |
| G90       | Absolute programming                 | Implemented in preamble() |
| G91       | Incremental programming              | Not directly implemented |

### M-Codes Comparison: Mach3 Manual vs GCodeWriter Implementation

| M-Code    | Mach3 Manual Function                | GCodeWriter Implementation          |
|-----------|--------------------------------------|-------------------------------------|
| M0        | Program stop                         | Not implemented                     |
| M1        | Optional program stop                | Not implemented                     |
| M3        | Spindle on (clockwise rotation)      | Not implemented                     |
| M4        | Spindle on (counterclockwise)        | Not implemented                     |
| M5        | Spindle stop                         | Not implemented                     |
| M6        | Tool change                          | Implemented in toolchange() function |
| M7        | Mist coolant on                      | Not implemented                     |
| M8        | Flood coolant on                     | Not implemented                     |
| M9        | All coolant off                      | Not implemented                     |
| M30       | Program end and rewind               | Not implemented                     |
| M47       | Repeat program from first line       | Not implemented                     |
| M48       | Enable speed and feed override       | Not implemented                     |
| M65       | Close shutter                        | Implemented in preamble() for Aerotech |
| M83       | Use relative distances for extrusion | Implemented for RepRap/Teacup/Repetier/Smoothie/Aerotech |
| M98       | Call subroutine                      | Not implemented                     |
| M99       | Return from subroutine/repeat        | Not implemented                     |

## Conversion table from Mach3-LinuxCNC to Aerotech

### Motion Commands
| Mach3-LinuxCNC      | Aerotech          | Description                                    |
|---------------------|-------------------|------------------------------------------------|
| G0                  | G0                | Rapid positioning                              |
| G1                  | G1                | Linear interpolation                           |
| G2                  | G2                | Clockwise circular interpolation               |
| G3                  | G3                | Counterclockwise circular interpolation        |
| G4                  | G4                | Dwell/Pause                                    |

### Plane Selection
| Mach3-LinuxCNC      | Aerotech          | Description                                    |
|---------------------|-------------------|------------------------------------------------|
| G17                 | G17               | XY plane select                                |
| G18                 | G18               | XZ plane select                                |
| G19                 | G19               | YZ plane select                                |

### Units and Position Mode
| Mach3-LinuxCNC      | Aerotech          | Description                                    |
|---------------------|-------------------|------------------------------------------------|
| G20                 | G70               | Set units to inches                            |
| G21                 | G71               | Set units to millimeters                       |
| G90                 | G90               | Absolute positioning                           |
| G91                 | G91               | Incremental positioning                        |
| G92                 | G92               | Set position                                   |

### Tool Compensation
| Mach3-LinuxCNC      | Aerotech          | Description                                    |
|---------------------|-------------------|------------------------------------------------|
| G40                 | G40               | Cancel cutter compensation                     |
| G41                 | G41               | Cutter compensation left                       |
| G42                 | G42               | Cutter compensation right                      |
| G43                 | G143              | Tool length compensation positive              |
| G49                 | G149              | Cancel tool length compensation                |

### Scaling and Work Offsets
| Mach3-LinuxCNC      | Aerotech          | Description                                    |
|---------------------|-------------------|------------------------------------------------|
| G50                 | G150              | Cancel scaling                                 |
| G51                 | G151              | Enable scaling                                 |
| G53                 | G53               | Disable work offsets                           |
| G54                 | G54               | Work offset 1                                  |
| G55                 | G55               | Work offset 2                                  |
| G56                 | G56               | Work offset 3                                  |
| G57                 | G57               | Work offset 4                                  |
| G58                 | G58               | Work offset 5                                  |
| G59                 | G59               | Work offset 6                                  |

### Motion Control
| Mach3-LinuxCNC      | Aerotech          | Description                                    |
|---------------------|-------------------|------------------------------------------------|
| G61                 | G9                | Exact stop mode                                |
| G64                 | G108              | Continuous mode/velocity blending              |
| G93                 | G93               | Inverse time feed mode                         |
| G94                 | G94               | Feed per minute mode                           |
| G95                 | G95               | Feed per revolution                            |

### Program Control
| Mach3-LinuxCNC      | Aerotech          | Description                                    |
|---------------------|-------------------|------------------------------------------------|
| M0                  | M0                | Program stop                                   |
| M1                  | M1                | Optional stop                                  |
| M2                  | M2                | Program end                                    |
| M48                 | M48               | Enable feed rate override                      |
| M49                 | M49               | Disable feed rate override                     |

### Spindle Control
| Mach3-LinuxCNC      | Aerotech          | Description                                    |
|---------------------|-------------------|------------------------------------------------|
| M3                  | M3/M103           | Spindle on clockwise                           |
| M4                  | M4/M104           | Spindle on counterclockwise                    |
| M5                  | M5                | Spindle stop                                   |

### 3D Printer Specific Commands
| Mach3-LinuxCNC      | Aerotech          | Description                                    |
|---------------------|-------------------|------------------------------------------------|
| M104 S[temp]        | M104 P[temp]      | Set extruder temperature                       |
| M109 S[temp]        | M109 P[temp]      | Set extruder temperature and wait              |
| M140 S[temp]        | M140 P[temp]      | Set bed temperature                            |
| M190 S[temp]        | M190 P[temp]      | Set bed temperature and wait                   |
| M106 S[speed]       | M106 P[speed]     | Set fan speed                                  | 
| M107                | M107              | Fan off                                        |
| M82                 | INCREMENTAL OFF   | Use absolute distances for extrusion           |
| M83                 | INCREMENTAL ON    | Use relative distances for extrusion           |
| G92 E0              | G92 A0            | Reset extruder position                        |
| G1 E[value]         | G1 A[value]       | Extrude material                               |
| G1 X[x] Y[y] E[e]   | G1 X[x] Y[y] A[e] | Move with extrusion                            |
| M84                 | DISABLE X Y Z A   | Disable motors                                 |
| G10                 | M65               | Disable extruder -> Disable shutter            |
| G11                 | M64               | Enable extruder -> Enable shutter              |

### Key Differences:
1. **Parameter Format**: 
   - Aerotech uses 'P' instead of 'S' for temperature and speed values
   - Extrusion axis is 'A' instead of 'E'
   - Comments use // instead of ;

2. **Motion Control**:
   - Aerotech offers more detailed motion control with G8/G9 commands
   - Additional acceleration control commands (G60-G69)
   - Velocity blending control with G108/G109

3. **Coordinate Systems**:
   - Both systems support similar work offset commands (G54-G59)
   - Aerotech adds more coordinate plane control options

4. **Special Features**:
   - Aerotech includes additional commands for scaling (G150/G151)
   - Enhanced spindle control with non-blocking commands (M103/M104)
   - More precise motion control with G359/G360/G361


   ## Prusa slicer Gcode flavors
      gcfRepRapSprinter
      gcfRepRapFirmware
      gcfRepetier
      gcfTeacup
      gcfMakerWare
      gcfMarlinLegacy
      gcfMarlinFirmware
      gcfKlipper
      gcfSailfish
      gcfMach3
      gcfMachinekit
      gcfSmoothie
      gcfNoExtrusion
      gcfAerotech