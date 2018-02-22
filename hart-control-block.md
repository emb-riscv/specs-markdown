# The Hart Control Block (HCB)

For uniform access by software, each hart maps its own status registers to the same address in the memory space.

## RISC-V compatibility registers

These registers are mandated by the RISC-V Volume I, Chapter 2.8, for the `rdcycle` and `rdinstret` instructions.

RV64 devices:

- `hcb.cyclecnt`: cycle count for `rdcycle` (64-bits)
- `hcb.instcnt`: instructions count for `rdinstret` (64-bits)

RV32 devices:

- `hcb.cyclecntl`: low word of cycle count for `rdcycle` (32-bits)
- `hcb.cyclecnth`: high word of cycle count for `rdcycleh` (32-bits)
- `hcb.instcntl`: low word of instructions count for `rdinstret` (32-bits)
- `hcb.instcnth`: high word of instructions count for `rdinstreth` (32-bits)

RV32/RV64 devices:

- `hcb.hartid`: the current hart ID (32-bits)

Other RISC-V registers from RISC-V Volume II, but not actively used:

- mstatus 
- mie 
- mtvec 
- mscratch 
- mepc 
- mcause 
- mtval 
- mip 
- misa 

## RISC-V microcontroller specific registers

- `hcb.interrupts[]`: array of words with status for each interrupt (32-bits)
- `hcb.exctab`: address of the exception table (xlen)
- `hcb.irqtab`: address of the interrupt table (xlen)
- `hcb.irqprio`: interrupt priority threshold (32-bits)
- `hcb.msp`: main stack poiner (xlen)
- `hcb.msplimit`: the lowest address the main stack can descend (xlen)
- `hcb.tsp`: thread stack poiner (xlen)
- `hcb.tsplimit`: the lowest address the thread stack can descend (xlen)

TODO: 

- add MPU registers
