# Memory map

Generally the RISC-V microcontroller memory map is implementation specific; the only reserved area 
is a slice of 256 MiB at the end of the memory space.

For 32-bits devices, the reserved system area is **0xF0000000-0xFFFFFFFF**.

For 64-bits devices, the reserved system area is **0xFFFFFFFF'F0000000-0xFFFFFFFF'FFFFFFFF**.

Typical RISC-V microcontroller devices have:

- a **read-only code** area, usually at 0x00000000, but the actual address may be implementation 
specific (typically flash)
- a **read-write data** area (typically RAM)
- an implementation specific **peripheral** area.

Multi-hart devices can share certain memory areas (code or data), but can also have hart-specific 
memory areas, or both shared and specific areas.


## The Hart Control Block (HCB)

For uniform access by software, each hart maps its own status registers to the same address in the memory space.

### RISC-V compatibility registers

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

### RISC-V microcontroller specific registers

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

## The Device Control Bloc (DCB)

The DCB includes system peripherals and other registers non specific to any hart.

### RISC-V compatibility registers:

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

### RISC-V microcontroller specific registers

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


