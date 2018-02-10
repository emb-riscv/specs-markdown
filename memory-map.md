# Memory map

The memory map is device specific, the only reserved area is a slice of 256 MiB at the end of the memory space.

For 32-bits devices, the reserved system area is 0xF0000000-0xFFFFFFFF.

For 64-bits devices, the reserved system area is 0xFFFFFFFF'F0000000-0xFFFFFFFF'FFFFFFFF.

Typical embedded RISC-V devices have:

- a flash area, usually at 0x00000000, but the actual address may be device-specific
- a RAM area 
- a device-specific peripheral area.

Multi-hart devices may share certain memory area, but can also have hart-specific flash or RAM, or both common and specific areas


## The Hart Control Block (HCB)

Each hart maps its own status registers to the same address in the memory space.

### RISC-V compatibility registers

RV32/RV64 devices:

- `hcb.cyclecnt`: cycle count for `rdcycle` (xlen)
- `hcb.instcnt`: instructions count for `rdinstret` (xlen)

RV32 devices also have:

- `hcb.cyclecnth`: high word of cycle count for `rdcycleh` (32-bits)
- `hcb.instcnth`: high word of instructions count for `rdinstreth` (32-bits)

### Embedded specific registers

- `hcb.interrupts[]`: array of words with status for each interrupt (32-bits)
- `hcb.exctab`: address of the exception table (xlen)
- `hcb.irqtab`: address of the interrupt table (xlen)
- `hcb.irqprio`: interrupt priority threshold (32-bits)
- `hcb.stklimit`: the lowest address the stack can descend (xlen)

## The Device Control Bloc (DCB)

The DCB includes system peripherals and other registers non specific to any hart.

### RISC-V compatibility registers:

RV32/RV64 devices:

- `dcb.rtclock.counter`: RTC timer count for `rdtime`

RV32 devices also have:

- `dcb.rtclock.counterh`: high word of RTC timer count for `rdtimeh` (32-bits)

### Embedded specific registers

RV32/RV64 devices:

- `dcb.sysclock.counter`: system clock counter (xlen)
- `dcb.sysclock.ctrl`: system clock control register
- `dcb.sysclock.current`: system clock current value register (32-bits)
- `dcb.sysclock.reload`: system clock auto reload register (32-bits)

- `dcb.rtclock.alarm`: RTC clock alarm comparator (xlen)

RV32 devices also have:

- `dcb.sysclock.counterh`: high word of system clock counter (32-bits)

- `dcb.rtclock.alarmh`: high word of RTC clock alarm comparator (32-bits)

TODO:

- add watchdog registers `dcb.wdog.*`.




