# DFC - Dumb Frame Comp
A gcode_macro based frame thermal expansion compensation for Klipper machines
Part of the [Thermal Expansion Compensation Package](https://github.com/Deutherius/TECPac).

# WARNING - DEPRECATED - DO NOT USE DFC
After weeks of dealing with random blobs on curved sections of my prints, I traced the cause to DFC. 

**TL;DR**: don't use it in its current form, delete it from your printer config (disabling it through the enable flag is *not* enough), forget this exists and go install the proper, [not-dumb version](https://github.com/alchemyEngine/klipper_frame_expansion_comp).

**Long version**: The current implementation injects a Z adjust offset command into the gcode queue roughly every 10 seconds. When this happens, the printer has to finish the Z adjust (basically a very tiny Z move) before moving in X or Y. This cannot happen to long straight walls, but curves are basically just a bunch of tiny straight walls. This means that there is a high likelyhood of this Z adjust move happening just after a long linear XY move, or at any point within a curve. Problem is that while this Z adjust move is imperceptible to human eye, it stops the toolhead for just enough time (I'm guessing a couple of ms, but don't quote me on that) to cause a tiny ooze due to nozzle overpessure, which manifests as a small blob. Increasing pressure advance and decreasing perimeter speed can lessen this artifact (less pressure in the nozzle), but will never completely fix it. The reason why setting the enable flag to 0 will not stop this from happening is that I am a potato and keep sending the Z adjust command with a 0 value. Apparently that is till enough to stop the toolhead and cause a blob.

**I am really sorry if this affected you.** It certainly caused me a lot of headaches and headscratching. Thanks to bythorsthunder for sticking with me and helping me troubleshoot this.

**Will there be a solution?** It is probably possible to solve this. I could probably intercept G0/G1 commands and inject the Z adjust at a better time (during a retract, in the sparse infill, during a travel move...) where the blob will at least not be visible. But even that has things that can go wrong (e.g. very long first layer with just perimeters and solid infill - this is exactly the place where DFC is most useful, but can cause the most damage). Or it might be possible to keep an internal Z offset and inject Z movements to what were previously XY-only moves - this type of a Z adjust would not cause the printhead to stop. 

That being said, sounds like a very overcomplicated and overkill solution for something that can be better solved using a Moonraker plugin. If you are wondering, the real frame comp likely does not suffer from any such issue, because it directly works with the transform subsystem which runs in parallel and can stack with other transforms like tilt and mesh. 

This was always a kind of a stopgap solution before the bug in frame comp I mention in the next sections got fixed. Just go with the real thing.

I will keep this repo up for anyone who wants to play with it, but do not recommend using it.

# Thermal WHAT?

If you have a reasonably-sized enclosed 3D printer with vertical aluminium extrusions, you might be suffering from, among other issues, thermal frame expansion. This repo provides an easy way of correction this issue. For the "other issues", such as real-time bimetallic gantry bowing compensation or temperature-dependent changing first layer Z offset, visit my other repos - former issue [here](https://github.com/Deutherius/VGB), latter issue [here](https://github.com/Deutherius/Gantry-bowing-induced-Z-offset-correction-through-relative-reference-index).

# Thermal Frame Expansion

As metal heats up, it expands. The frame is relatively well constrained, but still subject to some thermal expansion. In the case of a Voron 2.4, the Z extrusions expand, which manifests as an upwards Z drift with increasing frame temperature. This can be particularly noticeable if you start a full plate print job from a relatively cold state of the printer (i.e. bed and hotend at temp and heatsoaked, but frame still relatively cold). As the first layer prints, the frame expands. When the second layer starts, the toolhead is no longer where it is expected, but quite a bit higher, as is the case in the picture below.

![20210816_195655](https://user-images.githubusercontent.com/61467766/130957038-48d03ab3-b28a-44ac-b7cd-b1b4b34ed25f.jpg)

Left part is from a 7-hour print job with a first layer time of 22 minutes. The right part was printed alone, with a first layer of 1 minute and 45 seconds. All settings and printer state was the same (cold) for both prints. Both prints have roughly the same amount of Z error due to frame expansion, but in the right part it is much more spread out throughout the entire part, whereas in the left part, the error is mostly localized in the first few layers.

**Note that this issue is actually mostly caused by the aforementioned bimetallic gantry bowing, particularly in the middle of the print bed. Frame expansion is still at play though, and the image serves as a nice visual aid, which is why I use it in both repos. Refer to my other repos for more information on gantry bowing.**

The solution to this problem is the [Frame Expansion Compensation](https://github.com/alchemyEngine/klipper/tree/work-frame-expansion-20210410) fork of klipper by alchemyEngine. For more info, read up on [whoppingpochard's repo](https://github.com/tanaes/whopping_Voron_mods/tree/main/docs/frame_expansion) that describes how to set it up. I tried it out, it solves the frame expansion problem in a neat built-in fashion. In my case, however, there were bugs under the hood that I was not able to iron out - you can read the full report [here](https://github.com/Deutherius/FrameCompTesting). TL;DR: As the compensation value got bigger, Z motors started spazzing out, which manifested as visual artifacts in layers. Before alchemy was able to find and fix the bug, I developed a gcode_macro-based solution that works pretty much the same, but applies the compensation using Z_offset adjustments, which avoids any potential conflicts with klipper's innards.

To be completely clear, alchemy's version is now fixed and you should give it a go first, it's much more elegant. The "dumb" version you will find here does not have all the features (e.g. thermistor smoothing) and is only meant as a last resort, or if you need to remain on the main klipper branch for some reason.

# How to set DFC up and what exactly does it do

Included in this repo you will find a config file named `DFC.cfg`. To use it, simply copy the .cfg file into the same folder as your printer.cfg and add the following line anywhere in your printer.cfg:

`[include DFC.cfg]`

In addition to the above, you need to have a thermistor measuring frame temperature set up as `[temperature_sensor frame]` (the easy part), and have your frame expansion coefficient calculated (the harder part). Refer to [Whopping's repo](https://github.com/tanaes/whopping_Voron_mods/tree/main/docs/frame_expansion) on how to do that. You do *not* need to have the Frame Comp branch of klipper installed to run the measure_thermal_behavior.py script. You can specify a different thermistor name should you wish to either name your thermistor differently, or outright use a different thermistor placement (not recommended).

You will also need to insert these 3 lines at the last Z homing before a print:
```
  {% if printer['gcode_macro _DFC'] %}
     _LOG_REF_TEMP
  {% endif %}
```
The easiest way is to add these lines to the `[homing_override]` section in your printer.cfg. If you don't already use `[homing_override]`, feel free to use the following:

```
[homing_override]
gcode:
  G90
  {% if "xy" not in printer.toolhead.homed_axes %}
    G0 Z5 F1600
    G28 X Y
  {% endif %}
  G28 Z
  G0 Z10 F1800
  
  {% if printer['gcode_macro _DFC'] %}
     _LOG_REF_TEMP
  {% endif %}
```
The `if` statement around the macro call makes it so that you don't have to delete or comment out the line should you decide to temporarily (or permanently) disable or delete DFC.

Alternatively, just add the aforementioned lines after the last `G28` or `G28 Z` command before the print begins. This saves the current frame temperature every time you home your printer as a reference.

And finally, open DFC.cfg and set `variable_coeff: 0` to your calculated frame expansion coefficient (e.g. mine is quite high at 0.03 - if yours is higher, something might be wrong with your coefficient calculation or data collection). You might also want to set an absolute limit if you find that DFC is oversquishing your parts (i.e. the height of your part is significantly shorter than it should be) - this lets DFC compensate for frame expansion where it most matters, i.e. during the first few layers, but stops the compensation at yoru specified limit. Adapt `variable_max_abs_comp: 0` to your desired limit (0 means no limit). I personally run at 0.03 mm/K and 0.1 mm limit, but your value will obviously vary.

The main function is rather simple - every 10 seconds, a frame temperature measurement is taken and compared to a reference temperature. Their difference is multiplied by your expansion coefficient, and this value is the new negative Z offset. This adjustment is independent of the user-set Z offset - but beware, you should only use Z adjustment (i.e. live Z adjust during a print `SET_GCODE_OFFSET Z_ADJUST=0.1`), never "Z set" (i.e. offset reset `SET_GCODE_OFFSET Z=0.0`)! The macro has no way of checking if the Z offset has been changed outside of its influence and will continue to function as if it was not.

IMPORTANT! When a macro has an underscore in front of it, it is not meant to be called by the user. This underscore is a way of telling your web interface to not create a button for this macro. This works in Mainsail, not sure about other web interfaces (Fluidd, Octoprint...). In any case, if you see it, hide it. *However*, make sure to copy and paste the full name of the macro (including the underscore) when prompted to in the further parts of this guide.

That's it! You can query the DFC status by calling `QUERY_DFC`, which will output the current temps, coeff, enable flag, z offset and max abs limit into the console. You can also enable and disable DFC by calling `SET_DFC ENABLE=1` and `SET_DFC ENABLE=0` (**WARNING - disabling DFC returns its Z offset to 0. This can case a large Z jump during a print, and in certain conditions cause a nozzle strike! You should never turn DFC on or off during a print!**)

### SAFETY

Since this function directly changes the Z offset, there is a real possibility of a nozzle strike if something goes wrong. With reasonable values (coeff not overshot, thermistor stable, maximum limit set), this should not happen - but you should always keep your eyes on the Z offset and be ready to E-stop your printer if things get out of hand! Mistakes happen and they can become expensive rather fast (ask me how I know). You have been warned.

