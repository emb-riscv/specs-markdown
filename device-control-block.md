# The Device Control Bloc (`dcb`)

The DCB includes system registers that are not specific to a given hart.

## Memory map

| Offset | Name | Width | Description | 
|:-------|:-----|:------|-------------|
| 0xXXXX | hartidlast | 32b | the ID of the last hart in the device |
| 0xXXXX | harts[] | ?? | Array of hart status visible/accessible by all harts, like bits for pending interrupts |


## RISC-V compatibility registers:

The RISC-V Volume I, Chapter 2.8, mandates for the `rdtime` instruction. This can be retrieved from `dcb.rtclock.counter`.

Other RISC-V registers from RISC-V Volume II:

- mvendorid 
- marchid 
- mimpid 
- misa 

Other registers that migh need attention:

-required for the debug module: 
  - dcsr 
  - dpc 
- optional for the debug module: 
  - dscratch0 
  - dscratch1 


