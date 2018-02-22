# The Device Control Bloc (`dcb`)

The DCB includes system registers that are common to the entire device and are not specific to any given hart.

## Memory map

| Offset | Name | Width | Type | Description | 
|:-------|:-----|:------|:-----|-------------|
| 0x0000 | hartidlast | 32b | ro | The ID of the last hart in the device. |
| 0x0004 | | | | Reserved. |
| 0x0100 | harts[] | 256B x N | | Array of hart status and control. |

## ID of the last hart

For multi-hart devices, reading this register returns the ID of the last hart. Single-hart devices return 0.

## Hart status and control

For multi-hart devices, these registers allow one hart to pend interrupts to any other hart, and possibly to handle the priority thresholds, to handle synchronization issues like priority inversion.

> <sup>Use case must be further investigated.</sup>

For single-hart devices, this area is reserved.

| Offset | Name | Width | Type | Description | 
|:-------|:-----|:------|:-----|-------------|
| 0x0000 | pendings[] | 32b x 32 | w1s | Hart interrupt pending bits. |
| 0x0080 |  |  |  | Reserved. |
| 0x00FC | prio | 32b | rw | Hart priority threshold. |

Total size: 256 bytes.

TODO: Possibly protect writes using a key register.

## RISC-V compatibility CSRs

TODO: decide what to do with these registers; probably some of them should be added to `dcb`.

The RISC-V Volume I, Chapter 2.8, mandates for the `rdtime` instruction. This can be retrieved from `dcb.rtclock.counter`.

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
