# Supported Boards
---
![Boards](../images/boards.png)

While TinyCLR OS can be ported to other devices, we are providing a ready port for a few popular boards.

One of the ports is for FEZ, which is the official TinyCLR OS board. It will always have the latest and greatest and should be used as a reference for new ports.

# Firmware Update
The firmware needs to be updated to match the assemblies loaded by Visual Studio. If you see an error message about a checksum error then your project's assemblies version does not match the veriosn loaded on the device.

Each board will have details on how to load its firmware.

# Other Supported Devices
Several of the GHI Electronics' products support TinyCLR OS. Check the individual [hardware pages](../../hardware/intro.md) to see which devices support TinyCLR OS.

* [**FEZ**](fez.md)
* [**STM32 Boards**](stm32_boards.md)
* [**Netduino**](netduino.md)
* [**MikroElektronika**](mikroelektronika.md)
