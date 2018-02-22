# The Device Control Bloc (DCB)

The DCB includes system peripherals and other registers non specific to any hart.

## RISC-V compatibility registers:

These registers are mandated by the RISC-V Volume I, Chapter 2.8, for the `rdtime` instruction.

RV64 devices:

- `dcb.rtclock.counter`: RTC timer count for `rdtime` (64-bits)

RV32 devices:

- `dcb.rtclock.counterl`: low word of RTC timer count for `rdtime` (32-bits)
- `dcb.rtclock.counterh`: high word of RTC timer count for `rdtimeh` (32-bits)

Other RISC-V registers from RISC-V Volume II:

- mvendorid 
- marchid 
- mimpid 
- misa 

## RISC-V microcontroller specific registers

RV64 devices:

- `dcb.sysclock.counter`: system clock counter (64-bits)
- `dcb.rtclock.alarm`: RTC clock alarm comparator (64-bits)

RV32 devices:

- `dcb.sysclock.counterl`: low word of system clock counter (32-bits)
- `dcb.sysclock.counterh`: high word of system clock counter (32-bits)

- `dcb.rtclock.alarmh`: high word of RTC clock alarm comparator (32-bits)
- `dcb.rtclock.alarmh`: high word of RTC clock alarm comparator (32-bits)

RV32/RV64 devices:

- `dcb.sysclock.ctrl`: system clock control register (32-bits)
- `dcb.sysclock.current`: system clock current value register (32-bits)
- `dcb.sysclock.reload`: system clock auto reload register (32-bits)

- `dcb.hartidlast`: the ID of the last hart in the device (32-bits)
- `dcb.harts[]` hart status visible/accessible by all harts, like bits for pending interrupts.

TODO:

- add watchdog registers `dcb.wdog.*`.

Other registers that migh need attention:

-required for the debug module: 
  - dcsr 
  - dpc 
- optional for the debug module: 
  - dscratch0 
  - dscratch1 


