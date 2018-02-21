# The Device System Clock (DSC)

## Overview

The **Device System Clock** is intended to support the implementation of the ISO/IEC 14882.2011 
`high_resolution_clock` (§ 20.11.7.3). Objects of class `high_resolution_clock` represent clocks 
with the shortest tick period.

The **Device System Clock** is also intended to generate the scheduler interrupts.

All harts in a RISC-V device share the same Device System Clock instance.

The Device System Clock is inspired by the `mtime`/`mtimecmp` definitions in the RISC-V 
privileged specs, but it counts a higher frequency input.

## Power domain

The system clock is required to run only when the device is powered up, and so it can be 
located in the same frequency/voltage domain as the cores.

## Clock input

The system clock source is application or device specific.

> <sup>Common implementations use the same source as the core input, before possible PLLs. 
With a typical 10 MHz input, the clock resolution 
is 0.1 µS and it takes about 58000 years to overflow.</sup>

## The clock counter register

The system clock time point register is a 64-bit counter, common on all RV32 and RV64 devices.

To guarantee the steadiness characteristic of the clock, the register is read-only. 
(TODO: define the bit/mechanism that starts it)

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions. 
RV32 devices exposes separate high/low 32-bits registers.

RV64 devices:

- `dcb.sysclock.cnt` 

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [63-0] | `cnt` | ro | System clock timer counter (64-bits). |


RV32 devices:

- `dcb.sysclock.cntl`
- `dcb.sysclock.cnth`

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [31-0] | `cntl` | ro | Low word of system clock timer counter (32-bits). |
| [63-32] | `cnth` | ro | High word of system clock timer counter (32-bits). |

## The clock comparator register

The system clock can also be used to trigger periodic interrupts, for example 
to drive pre-emption in a scheduler. 

The comparator register causes a system clock interrupt to be posted when the 
counter register 
contains a value greater than or equal to the value in the comparator register.
The interrupt remains posted until it is cleared by writing to the comparator register.

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions. 
RV32 devices exposes separate high/low 32-bits registers.

RV64 devices:

- `dcb.sysclock.cmp` 

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [63-0] | `cmp` | rw | System clock timer comparator (64-bits). |


RV32 devices:

- `dcb.sysclock.cmpl`
- `dcb.sysclock.cmph`

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [31-0] | `cmpl` | ro | Low word of system clock timer comparator (32-bits). |
| [63-32] | `cmph` | ro | High word of cycem clock timer comparator (32-bits). |

## The clock status and control register

- `dcb.sysclock.ctrl`

TODO: define status and control bits.
