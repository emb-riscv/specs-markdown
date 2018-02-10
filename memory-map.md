# Memory map

The memory map is device specific, the only reserved area is a slice of 256 MiB at the end of the memory space.

For 32-bits devices, the system space is 0xF0000000-0xFFFFFFFF.

For 64-bits devices, the system space is 0xFFFFFFFF'F0000000-0xFFFFFFFF'FFFFFFFF.

Typical devices have:

- a flash area, usually at 0x00000000, but the actual address may be device-specific
- a RAM area 
- a device-specific peripheral area.

Multi-hart devices may share certain memory area, but can also have hart-specific flash or RAM, or both common and specific areas


## The Hart Control Block (HCB)

Each hart maps its own status registers to the same address in the memory space.

### RISC-V compatibility registers

RV32/RV64 devices:

- `cyclecnt`: cycle count for `rdcycle` (xlen)
- `instcnt`: instructions count for `rdinstret` (xlen)

RV32 devices also have:

- cyclecnth: high word of cycle count for `rdcycleh` (32-bits)
- instcnth: high word of instructions count for `rdinstreth` (32-bits)

### Embedded specific registers

- `interrupts[]`: array of words with status for each interrupt (32-bits)
- `exctab`: address of the exception table (xlen)
- `irqtab`: address of the interrupt table (xlen)
- `irqprio`: interrupt priority threshold (32-bits)
- `stklimit`: the lowest address the stack can descend (xlen)


## The Device Control Bloc (DCB)

The DCB includes system peripherals and other registers non specific to any hart.

### RISC-V compatibility registers:

RV32/RV64 devices:

- `rtclock.counter`: RTC timer count for `rdtime`

RV32 devices also have:

- `rtclock.counterh`: high word of RTC timer count for `rdtimeh` (32-bits)

### Embedded specific registers

RV32/RV64 devices:

- `sysclock.counter`: system clock counter (xlen)
- `sysclock.ctrl`: system clock control register
- `sysclock.current`: system clock current value register (32-bits)
- `sysclock.reload`: system clock auto reload register (32-bits)

- `rtclock.alarm`: RTC clock alarm comparator (xlen)

RV32 devices also have:

- `sysclock.counterh`: high word of system clock counter (32-bits)

- `rtclock.alarmh`: high word of RTC clock alarm comparator (32-bits)

TODO:

- add watchdog registers



