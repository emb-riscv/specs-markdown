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
| 0x0004 | | | | | Reserved.  |
| 0x0008 | | | | | Reserved.  |
| 0x00F0 | `cyclecntl` | 32b | ro | 0x00000000 | Low word of cycle count. |
| 0x00F4 | `cyclecnth` | 32b | ro | 0x00000000 | High word of cycle count. |
| 0x00F8 | `instcntl` | 32b | ro | 0x00000000 | Low word of instructions count. |
| 0x00FC | `instcnth` | 32b | ro | 0x00000000 | High word of instructions count. |

TODO: add the `sysclock.cmp` here.
