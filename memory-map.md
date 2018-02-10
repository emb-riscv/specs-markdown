# Memory map

The memory map is device specific, the only reserved area is a slice of 256 MiB at the end of the memory space.

For 32-bits devices, the system space is 0xF0000000-0xFFFFFFFF.

For 64-bits devices, the system space is 0xFFFFFFFFF0000000-0xFFFFFFFFFFFFFFFF.

## The Hart Control Block (HCB)

Each hart maps its own status registers to the same address in the memory space.

### RISC-V compatibility registers

RV32/RV64 devices:

- `cyclecnt`: cycle count for `rdcycle`
- `instcnt`: instructions count for `rdinstret`

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

- `rtclock`: timer count for `rdtime`

RV32 devices also have:

- `rtclockh`: high word of timer count for `rdtimeh` (32-bits)

### Embedded specific registers

RV32/RV64 devices:

- `sysclock`: system clock counter (xlen)
- 

RV32 devices also have:

- `sysclockh`: high word of system clock counter (32-bits)




