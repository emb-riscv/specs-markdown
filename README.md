# The RISC-V Microcontroller Profile

A proposal for a friendlier microcontroller architecture using the beautiful RISC-V instruction set.

Version: 0.1

Editors:
* Liviu Ionescu

Warning: This draft specification is in a preliminary phase and may change at any time. For the moment it is more like a wish list than a real specs document.


## Table of Contents

* [Introduction](introduction.md)
* [Memory Map](memory-map.md)
* [The Startup Process](startup.md)
* [Exceptions and Interrupts](exceptions-and-interrupts.md)
* [Hart Control Block (`hcb`)](hart-control-block.md)
* [Hart Interrupt Controller (`hic`)](interrupt-controller.md)
* [Device Control Block (`dcb`)](device-control-block.md)
* [Device System Clock (`sysclock`)](system-clock.md)
* [Device Real-Time Clock (`rtclock`)](real-time-clock.md)
* [Device Watchdog Timer (`wdog`)]()
* [RTOS Support Features](rtos-support-features.md)
* [Appendix A: Improvements upon RISC-V privileged](improvements-upon-privileged.md)
* [Appendix B: History](history.md)

TODO: Add MPU definitions

TODO: Add user mode.

## License

This document is released under a [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/legalcode) license.
