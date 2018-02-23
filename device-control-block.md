# The Device Control Bloc (`dcb`)

The DCB includes system registers that are common to the entire device and are not 
specific to any given hart.

## Memory map

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | hartidlast | 32b | ro | 0x00000NNN | The ID of the last hart in the device. |
| 0x0004 | | | | | Reserved. |
| 0x0100 | harts[] | 256B x N | | | Array of hart status and control. |

## ID of the last hart

For multi-hart devices, reading this register returns the ID of the last hart available 
in the device. Single-hart devices return 0.

## Hart status and control

For multi-hart devices, these registers allow one hart to pend interrupts to any other 
hart, and possibly to temporarily adjust the priority thresholds, to handle synchronization 
issues like priority inversion.

> <sup>Use case must be further investigated.</sup>

For single-hart devices, this area is reserved.

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `pendings[]` | 32b x 32 | w1s | 0x00000000 | Hart interrupt pending bits. |
| 0x0080 | | | | | Reserved. |
| 0x00F8 | `prioth` | 32b | rw | 0x00000000 | Hart priority threshold. |
| 0x00FC | `key` | 32b | w | 0x00000000 | Access key. |

Total size: 256 bytes.

The hart status and control area has one bit of state. To prevent inadvertent interrupt 
pendings, all writes to this area (`pendings[]` and `prioth`) must be preceded by an 
unlock operation to the `key` register. The value (0x51F15000 + (Hart ID)) must be 
written to the `key` register to set the state bit before any write access to any 
other hart status and control register. The state bit is cleared at reset, and after 
any write to `pendings[]` or `prioth` registers. The `prioth` registers may be read 
without setting the `key`.

The `pendings[]` array packs `pending` bits for 32 interrupts in each element. 
The pending bit for interrupt N is stored in bit (N mod 32) of word (N/32). 
When 1 is written, the corresponding `pending` status bit is set.

TODO: The key mechanism has sychronization problems in case multiple harts access it 
simultaneously. Search for alternate solutions.

TODO: check if it is cheaper to use a single register with a value representing the 
number of the interrupt to pend, same as ARM STIR (Software Triggered Interrupt Register).

## RISC-V compatibility CSRs

TODO: decide what to do with these registers; probably some of them should be 
added to `dcb`.

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
