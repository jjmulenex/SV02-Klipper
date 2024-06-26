Klipper on the Sovol SV02 with Dual Extrusion (Orca and Cura)
-------------------------------------------------------------

### Introduction

This guide details setting up Klipper on the Sovol SV02 for dual extrusion, allowing you to use both Orca and Cura slicers seamlessly.

Orca's filament change differs from Cura. While Cura allows extruder start/end gcode, Orca uses gcode per filament. Duplicating filaments in Orca to mimic Cura seems impractical.

I used the Change filament gcode sestion under printer settings to do the job. This utilizes a macro named `COMPARE_EXTRUDER` to verify a switch between extruders is happening. If a new extruder is indeed selected (`current_extruder `doesn't match `next_extruder`), the `END_EXTRUDER` macro is called to perform any necessary actions for the previous extruder before switching.

I also created a Macro to hold a variable that I could use to detect if the extruder had been loaded yet. The first time Cura runs `START_EXTRUDER` the macro `FILAMENT_LOADED` is called and the variable_filament_loaded is set to True only the first time that Cura calls the extruder start gcode. This allows the loading filament at the start of the print and call the `LINE_PURGE` macro from KAMP. The next time Then switches to the `next_extruder` (only in Orca) and uses a `purge_bucket` macro to flush the nozzle

**Note** Since this gcode is called before a filament change but not before.  No filament loading on start. Hince the Filament loading (Orca only) decision in the `START_PRINT` gcode to solve the loading problem.

All of these macros are in the printer.cfg in this repo

### Goals

-   Maintain Cura start gcode without modification.
-   Enable printing from either extruder.
-   Consolidate code for efficiency.


###  Printer / Slicer

-   Klipper on a Raspberry PI 3B
-   BLTouch
-   2 in 1 out Stock Hotend
-   [Cura Settings](#cura-settings)
-   [Orca Settings](#orca-settings)


### Macros Used
- [START_PRINT](#start_print)
- [START_EXTRUDER](#start_extruder)
- [END_EXTRUDER](#end_extruder)
- [COMPARE_EXTRUDER](#compare_extruder-orca)
- [PURGE_BUCKET](#purge_bucket-orca-only-for-now)

#### START_PRINT 

Macro to prepare and start a print
Both Cura and Orca use this macro. There is a `SLICER` param to define the extruder prep method.

**Orca Settings** 

![Orca Start Settings](_images/start-orca.png)

**Cura Settings**

![Cura Start Settings](_images/start-orca.png)

To be more specific I needed to create a way to only load the filiment if Orca was used. In Orca settings we only define a tool change macro in the tool change macro section of settings. So each print would start without the filiment loaded. This part of the macro preloads the filament accordingly for any print sliced with Orca. 


##### Slicer gcode
```
START_PRINT EXTRUDER_TEMP=[first_layer_temperature] BED_TEMP=[first_layer_bed_temperature] START_EXTRUDER=[initial_extruder]  PRINT_TYPE="orca" 
```

##### Macro

```
[gcode_macro START_PRINT]  ; Macro to prepare and start a print
gcode:
  # Set default values (optional, adjust in Klipper config)
  {% set BED_TEMP = params.BED_TEMP|default(50)|float %}
  {% set EXTRUDER_TEMP = params.EXTRUDER_TEMP|default(200)|float %}
  {% set INITIAL_EXTRUDER = params.START_EXTRUDER|default(0)|int %}
  {% set SLICER = params.PRINT_TYPE|default("default")|string %}
  
  # Reset filament loaded flag
  SET_GCODE_VARIABLE MACRO=FILAMENT_LOADED VARIABLE=filament_loaded VALUE=False

  # Units and fans
  G21 ; mm units
  M107 ; Turn off fans

  # Home and heat
  G28 ; Home axes
  M118 Bed: {BED_TEMP} Extruder: {EXTRUDER_TEMP}
  M140 S{BED_TEMP} ; Heat bed
  M109 S{EXTRUDER_TEMP} ; Heat extruder
  M190 S{BED_TEMP} ; Wait for bed temp

  # Bed mesh calibration (if defined elsewhere)
  BED_MESH_CALIBRATE

  # Select initial extruder
  T{INITIAL_EXTRUDER}

  # Filament loading (Orca only)
  {% if SLICER == "orca" %}
    M118 Loading filament for Orca
    LOAD_FILAMENT
    # M83 ; Relative extrusion mode
    # G1 Z5 F240 ; Move to Z height
    # G1 X10 Y20 F3000 ; Move to position
    # G1 E93 F2000 ; Extrude filament (relative)
    # G92 E0 ; Reset to absolute mode
    LINE_PURGE
  {% else %}
    M118 You are using CURA, Nothing to do... Continue
    # No specific actions for Cura (placeholder)
  {% endif %}
  
```  

#### START_EXTRUDER
Macro For Cura to load Filament and change temp if needed. Cura uses this macro because it does not have a tool change gcode section in settings. 

As a matter of fact I think it's important to mention that the `START_EXTRUDER` gcode is always called. The End gocde is **only** called when the slicer is processing more that one filament in a print.

##### Slicer gcode
I pass in a hard value of the extruder 0 for the left extruder and 1 for the right. 
```
START_EXTRUDER EXTRUDER=0 EXTRUDER_TEMP={material_print_temperature}
```

##### Macro
```
[gcode_macro START_EXTRUDER]  ; Select and initiate extruder
gcode:
  {% set EXTRUDER_TEMP = params.EXTRUDER_TEMP|default(200)|float %} 
  {% set EXTRUDER_NR = params.EXTRUDER|default(0)|int %}  

  M118 Loading Extruder T{EXTRUDER_NR}...
  G92 E0 ; Reset extruder distance
  G1 F2000 E93 ; Load filament
  G92 E0 ; Reset extruder distance again
  M104 S{EXTRUDER_TEMP} ; Set extruder temperature
  ```
#### END_EXTRUDER
Macro to move out of the way and unload Filament.

##### Slicer gcode
```
[gcode_macro END_EXTRUDER]
gcode:
  M118 Changing Filament...
  G92 E0 ;reset extruder distance
  G1 F1500 E-5 ;short retract
  G1 F2400 X250 Y220
  G1 F3000 E-93  ;long retract for filament removal
  G92 E0 ;reset extruder distance
  G90
  ```

### COMPARE_EXTRUDER (Orca)
So in Orca I needed a way to change filiment but it's a bit different than Cura.

**Orca Settings**

![Orca Change Filament](_images/change-orca.png)

##### Slicer gcode
```
COMPARE_EXTRUDER CURRENT_EXTRUDER=[current_extruder] NEXT_EXTRUDER=[next_extruder]
```

##### Macro
```
[gcode_macro COMPARE_EXTRUDER]
gcode:
  {% set current = params.CURRENT_EXTRUDER|default(0)|int %}  # Get passed Current extruder or use current
  {% set next = params.NEXT_EXTRUDER|default(0)|int %}  # Get passed Next extruder or use current
  {% if current != next %}
    M118 Switching Extruders to T{next}
    END_EXTRUDER
    T{next}
    PURGE_BUCKET
  {% else %}
     M118 Extruders are the same, skipping filament change... 
     T{current}
  {% endif %}
  ```


### PURGE_BUCKET (Orca only, for now)

I didnt not write this, but I can not remember where I got it. If someone knows who wrote this I will give attrubution. Its a replacement for the prime tower and seems to work well. So far I only call it in the compare extruder macro

##### Macro

```
[gcode_macro PURGE_BUCKET]
gcode:
 #Absolute Coordinates
 G90
 #Move to Scrape Area
 G0 X310 F3000
 #Use Relative Extrusion
 M83
 #Extrude amount for scrape
 G1 E93 F2000
 G1 E25 F100
 G1 E5 F85
 #Quick Retract to stop filament ooze
 G1 E-7 F1500
 #wait 3 seconds to allow for ooze
 G4 P3000
 #Scrape Material
 G0 X140 F8000
 #Realtive Coordinates
 G91
 #Lower Z
   # G1 Z-5 F480 # remove me
 #Wait for current moves to finish
 M400 
 #Extrude Amount Ready for Print
 G1 E6 F450
 #Reset Extruder
 G92 E0
 #Use Absolute Coordinates
 G90

```

### Cura Settings 
***

##### Printer Start gcode
```
; Start Klipper Macro
START_PRINT EXTRUDER_TEMP={material_print_temperature_layer_0} BED_TEMP={material_bed_temperature_layer_0} START_EXTRUDER={initial_extruder_nr}
```

##### Printer End gcode

```
; machine end code >>
END_PRINT
; << machine end code
```
##### Left Extruder Start gcode
```
; extruder start code >>
START_EXTRUDER EXTRUDER=0 EXTRUDER_TEMP={material_print_temperature}
; << extruder start code
```

##### Left Extruder End gcode
```
; extruder 1 end code >>
END_EXTRUDER
; << extruder 1 end code
M118 End Filiment Extruder T{extruder_nr}
```

##### Right Extruder End gcode
```
; extruder 2 end code >>
START_EXTRUDER EXTRUDER=1 EXTRUDER_TEMP={material_print_temperature}
; << extruder 2 end code
```

##### Right Extruder End gcode
```
; extruder 2 end code >>
END_EXTRUDER	
; << extruder 2 end code
```

### Orca Settings 
***

##### Machine Start gcode
```
START_PRINT EXTRUDER_TEMP=[first_layer_temperature] BED_TEMP=[first_layer_bed_temperature] START_EXTRUDER=[initial_extruder]  PRINT_TYPE="orca" 
```

##### Machine End gcode
```
END_PRINT
```

##### Change Filament gcode
```
COMPARE_EXTRUDER CURRENT_EXTRUDER=[current_extruder] NEXT_EXTRUDER=[next_extruder]
```
