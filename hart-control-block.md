# The Hart Control Block (HCB)

For uniform access by software, each hart maps its own status registers to the same address in the memory space.

## Memory Map

### RV64 devices

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `excvta` | 64b | rw | Startup | Exceptions vector table address.  |
| 0x0008 | `intvta` | 64b | rw | 0x00000000'00000000 | Interrupts vector table address.  |
| 0x0010 | | | | | Reserved.  |
| 0x00F0 | `cyclecnt` | 64b | ro | 0x00000000'00000000 | Cycle count. |
| 0x00F8 | `instcnt` | 64b | ro | 0x00000000'00000000 | Instructions count. |

### RV32 devices

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `excvta` | 32b | rw | Startup | Exceptions vector table address.  |
| 0x0004 | | | | | Reserved.  |
| 0x0008 | `intvta` | 32b | rw | 0x00000000 | Interrupts vector table address.  |
| 0x000C | | | | | Reserved.  |
| 0x0010 | | | | | Reserved.  |
| 0x00F0 | `cyclecntl` | 32b | ro | 0x00000000 | Low word of cycle count. |
| 0x00F4 | `cyclecnth` | 32b | ro | 0x00000000 | High word of cycle count. |
| 0x00F8 | `instcntl` | 32b | ro | 0x00000000 | Low word of instructions count. |
| 0x00FC | `instcnth` | 32b | ro | 0x00000000 | High word of instructions count. |

TODO: add the `sysclock.cmp` here.

## Exceptions vector table address (`excvta`)

The address of the exceptions dispatch table. The table is an array of addresses (xlen size elements) pointing to exception handlers (C/C++ functions).

The register is initialised with the value in the startup block.

If not set (i.e. 0x0) and an interrupt occurs, an exception is triggered (TODO: what exception?).

## Interrupts vector table address (`intvta`)

The address of the interrupts dispatch table. The table is an array of addresses (xlen size elements) pointing to interrupt handlers (C/C++ functions).

If not set (i.e. 0x0) and an interrupt occurs, an exception is triggered (TODO: what exception?).
