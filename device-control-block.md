# The Device Control Bloc (`dcb`)

The DCB includes system registers that are common to the entire device and are not specific to any given hart.

## Memory map

| Offset | Name | Width | Type | Description | 
|:-------|:-----|:------|:-----|-------------|
| 0x0000 | hartidlast | 32b | ro | The ID of the last hart in the device. |
| 0x0004 | | | | Reserved. |
| 0x0100 | harts[] | 256B x N | | Array of hart status and control. |

## Hart status and control

These registers allow one hart to pend interrupts to any other hart, and possibly to handle the priority thresholds, to handle synchronization issues ike priority inversion.

| Offset | Name | Width | Type | Description | 
|:-------|:-----|:------|:-----|-------------|
| 0x0000 | pendings[] | 32b x 32 | w1s | Hart interrupt pending bits. |
| 0x0080 |  |  |  | Reserved. |
| 0x00FC | prio | 32b | rw | Hart priority threshold. |

Total size: 256 bytes.

## RISC-V compatibility CSRs:

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

TODO: decide what to do with these registers; probably some of them should be added to `dcb`.
