# A Klipper plugin for a self calibrating z offset

This is a Klipper plugin to self calibrate the nozzle offset to the print surface on a Voron V2.
There is no need for a manual z offset or first layer calibration any more. It is possible to change any variable in the printer from the temperature,
the nozzle, the flex plate, any modding on the print head or bed or even changing the z endstop position value of the klipper configuration.
Any of these changes or even all of them together do **not** influence the first layer at all. It's amazing!

## Why this

With the Voron V2 z-endstop, you can exchange nozzles without changing the offset. And by using a mag-probe (or SuperPinda, but it's not probing
directly the surface) as a z-endstop, you can exchange the flex plates without changing the offset. But you cannot get both and even more.

And this is what I did. I just combined these two methods to be completely independent of any offset calibrations.

## Requirements

- A z-endstop where the tip of the nozzle drives on a switch (like the standard Voron V2 enstop)
- A magnetic switch based probe at the print head - instead of the stock inductive probe ([like the one from Annex](https://github.com/Annex-Engineering/Annex-Engineering_Other_Printer_Mods/tree/master/VORON_Printers/VORON_V2dot4/Afterburner%2BMagnetic_Probe_X_Carriage_Dual_MGN9))
- The switch of the probe needs to be used as normaly closed. The plugin is then able to detect that the mag-probe is attached to the print head.
- The `z_calibration.py` file needs to be copied to the `klipper/klippy/extras` folder.
- (My previous Klipper macro for compensating the temperature based expansion of the z-endstop pin is **not** needed anymore.)

## What it does

1. After normal homing of all axes, the z-endstop is used to determine the z-positions of the tip of the nozzle and the enclosure of the attached mag-probe switch (near the trigger pin) by driving them on the endstop and saving their positions.
2. Then the mag-probe is used to probe a point on the print surface.
3. With the first two positions, the distance in z between probe and nozzle is calculated.
4. The z-offset of the nozzle is then calculated by subtracting this distance from the probed high on the print surface.
5. The calculated offset is applied by using the `SET_GCODE_OFFSET` command.

The only downside is, that the trigger point cannot be probed directly. This is why the body of the switch is clickt on the endstop. This small offset can be taken from the datasheet of the switch and is not influenced in any way.

Temperature or humindity changes are not a problem since the switch is not influenced much by them and all values are probed in a small time period and only the releations to each other are used.

It even doesn't matter what z-endstop position is configured in Klipper. All positions are relative to this point - only the absolute values are different. But, it is advisable to configure a safe value here to not crash the nozzle in the build plate by accident. The plugin only changes the GCode Offset and it is still possible to move the nozzle over this offset.

The output of the calibration looks like this (the offset is the one which is applied as GCode offset):

```
Z-CALIBRATION: ENDSTOP=-0.300 NOZZLE=-0.300 SWITCH=6.208 PROBE=7.013 --> OFFSET=-0.170
```

The endstop value is the homed z position which is the same as the configure z-endstop position.

## How to configure it

The configuration of this plugin looks like this:

```
[z_calibration]
switch_offset: 0.675  # using SSG-5H
speed: 80             # the moving speed in x and y
probe_nozzle_x: 206   # coordinates for clicking the nozzle on the z-endstop
probe_nozzle_y: 300
probe_switch_x: 211   # coordinates for clicking the probe's switch on the z-endstop
probe_switch_y: 281
probe_bed_x: 150      # coordinates for probing on the print surface (the center point)
probe_bed_y: 150
```

The `switch_offset` is the already mentioned offset from the switch body (which is the probed position) to the actual trigger point. The value can be taken from the datasheet of the Omron switch (D2F-5: 0.5mm and SSG-5H: 0.7mm). It is good to start with a little less depending on the squishiness you prefer for the first layer. This value is a really fixed one!

## How to use it

The calibration is started by using the `CALIBRATE_Z` command. If the probe is not attached to the print head, it will abort the calibration process.
So, macros can help here to unpark and park the probe like this:

```
[gcode_macro CALIBRATE_Z]
rename_existing: BASE_CALIBRATE_Z
gcode:
    CG28
    M117 Z-Calibration..
    _SET_LOWER_STEPPER_CURRENT  # I lower the stepper current for homing and probing stuff - this is not needed 
    _GET_PROBE                  # a macro for fetching the probe first
    BASE_CALIBRATE_Z
    _PARK_PROBE                 # and parking it afterwards
    _RESET_STEPPER_CURRENT
    M117
```

Then the `CALIBRATE_Z` command needs to be added to the `PRINT_START` macro. For this, just replace the second z homing after QGL with this macro. The sequence could be like this:

1. home all axes
2. heat up the bed and nozzle
3. get probe, make QGL, park probe
4. clean the nozzle
5. CALIBRATE_Z
6. print intro line
7. start printing...

**!! Happy Printing with an always perfect first layer - doesn't matter what you just modded on your print head/bed or what nozzle and flex plate you like to use for the next print. It just stays the same :-) !!**

## Dislaimer

It works perfectly for me. But, at this moment it is not widely tested. And I don't know much about the Klipper internals. So, I had to figure everything out by myself and found this as a working way for me. If there are better/easier ways to accomplish it, please don't hesitate to contact me!