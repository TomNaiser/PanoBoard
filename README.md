# PanoBoard
Integrated Electronics Board for a Gigapixel Panorama Head (pan and tilt motion)
-Arduino nano based controller
-bluetooth communication with the windowsbased control notebook
-PID position control (modified RC-Servos)
-rotatory position encoders for precise position control 

The latest version PanoBoardV5 is a size-reduced version. With the shrunken electronics box the panorama rig fits into my photo backpack. 

## Arduino Nano R4 upgrade

The [PanoBoard V5 Nano R4 patch](PanoBoard5KiCad/Nano-R4-Patch.md) explains the resistor changes, GPIO current limits, encoder I²C pull-ups, power supply and firmware porting needed to replace the classic Nano with a Nano R4. The KiCad files remain the original V5 design; apply the documented rework before operating the R4.

Current software: [PanoDuinoControl firmware](https://github.com/TomNaiser/PanoDuinoControl), [Android app and system documentation](https://github.com/TomNaiser/PanoramaSoftware), and [PanoramaControl for Windows](https://github.com/TomNaiser/PanoramaControl).

Warning! The circuits in this project were designed as a hobbyist project and may contain unintentional mistakes.  I don't take any responsibility for damages caused by using the above circuit design. Please use the provided information at your own risk!

