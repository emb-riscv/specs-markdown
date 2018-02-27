# The Device Control Bloc (`dcb`)

The DCB includes system registers that are common to the entire device and are not 
specific to any given hart.

## Memory map

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | hartidmax | 32b | ro | 0x00000NNN | The highest value for hart ID in the device. |
| 0x0004 | vendorid | 32b | ro |  | Vendor ID. |
| 0x0008 | archid | 32b | ro |  | Architecture ID. |
| 0x000C | impid | 32b | ro |  | Implementation ID. |
| 0x0010 | | | | | Reserved. |
| 0x0020 | dcsr | | | | Debug CSR. |
| 0x0028 | dpc | | | | Debug PC. |
| 0x0030 | | | | | Reserved. |
| 0x0100 | harts | | | | All harts interrupts. |

## The highest hart ID

For multi-hart devices, reading this register returns the highest numerical value used as hart ID 
in the device. Single-hart devices must return 0.

## All harts interrupts

For multi-hart devices, these registers allow one hart to pend interrupts to any other 
hart, and possibly to temporarily adjust the priority thresholds, to handle synchronization 
issues like priority inversion.

> <sup>Use case must be further investigated.</sup>

For single-hart devices, this area is reserved.

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `hartid` | 32b | w | 0x00000000 | Access key. |
| 0x0004 | `pendnum` | 32b | w |  | Hart interrupt pending bits. |
| 0x0008 | `prioth` | 32b | rw | 0x00000000 | Hart priority threshold. |
| 0x000C |  |  |  |  | Reserved. |

These registers have one additional bit of state. To prevent inadvertent interrupt 
pendings, all writes to this area (`pendnum` and `prioth`) must be preceded by an 
unlock operation to the `hartid` register. The value (0x51F15000 + (Hart ID)) must be 
written to the `hartid` register to set the hart id and the state bit before 
any write access to `pendings` and `prioth`. 
The state bit is cleared at reset, and after any write to `pendnum` or `prioth` registers.

The `pendnum` register is write only and allows access to the hart identified 
by the write to `hartid`. Writing a small integer value pends the 
interrupt with the given number. The `pendnum` register must have enough bits to 
represent any interrupt number (at most 10).

The `prioth` register is read/write and allows access to the hart identified 
by the write to `hartid`. It must have enough bits to represent any interrupt
priority.

Warning: The key mechanism has sychronization problems in case multiple harts access it 
simultaneously. Implementations can choose to allow access only from hart 0.

## RISC-V compatibility CSRs

The RISC-V Volume I, Chapter 2.8, mandates for the `rdtime` instruction. This can be 
retrieved from `dcb.rtclock.counter`.

Other RISC-V registers from RISC-V Volume II:

- mvendorid 
- marchid 
- mimpid 

Other registers that migh need attention:

- required for the debug module: 
  - dcsr 
  - dpc 
- optional for the debug module: 
  - dscratch0 
  - dscratch1 
