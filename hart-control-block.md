# The Hart Control Block (HCB)

For uniform access by software, each hart maps its own status registers to the same address in the memory space.

## Memory Map

### RV64 devices

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `excvta` | 64b | rw | 0x00000000'00000000 | Exceptions vector table address.  |
| 0x0008 | | | | | Reserved.  |
| 0x00F0 | `cyclecnt` | 64b | ro | 0x00000000'00000000 | Cycle count. |
| 0x00F8 | `instcnt` | 64b | ro | 0x00000000'00000000 | Instructions count. |

### RV32 devices

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `excvta` | 32b | rw | 0x00000000 | Exceptions vector table address.  |
| 0x0008 | | | | | Reserved.  |
| 0x00F0 | `cyclecntl` | 32b | ro | 0x00000000 | Low word of cycle count. |
| 0x00F4 | `cyclecnth` | 32b | ro | 0x00000000 | High word of cycle count. |
| 0x00F8 | `instcntl` | 32b | ro | 0x00000000 | Low word of instructions count. |
| 0x00FC | `instcnth` | 32b | ro | 0x00000000 | High word of instructions count. |


## RISC-V microcontroller specific registers

- `hcb.msp`: main stack poiner (xlen)
- `hcb.msplimit`: the lowest address the main stack can descend (xlen)
- `hcb.tsp`: thread stack poiner (xlen)
- `hcb.tsplimit`: the lowest address the thread stack can descend (xlen)

## RISC-V compatibility CSRs

These registers are mandated by the RISC-V Volume I, Chapter 2.8, for the `rdcycle` and `rdinstret` instructions.

RV32/RV64 devices:

- `mhartid`: the current hart ID (32-bits)

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


TODO: 

- add MPU registers
