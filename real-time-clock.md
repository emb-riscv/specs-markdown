# The Device Real-Time Clock (DRTC)

## Overview

The **Device Real-Time Clock** is intended to support the implementation of the ISO/IEC 14882.2011 
`system_clock` (§ 20.11.7.1) and `steady_clock` (§ 20.11.7.2) classes. Objects of class 
`system_clock` represent wall clock time from the system-wide realtime clock. Objects of 
class `steady_clock` represent clocks for which values of the time point never decrease as 
physical time advances and for which values of time_point advance at a steady rate 
relative to real time. That is, the clock may not be adjusted.

All harts in a RISC-V device share the same Device Real-Time Clock instance.

The Device Real-Time Clock is inspired by the `mtime`/`mtimecmp` definitions in the RISC-V privileged specs.

## Power domain

To support full functionality, the real-time clock should run even when the 
rest of system is powered down, and so it should be located in a different frequency/voltage 
domain from the cores.

## Clock input

The real-time clock input frequency is fixed, and application or device specific.

To support low-power devices, the real-time clock input should be a low freqency oscilator; the actual 
source is implementation dependent. 

> <sup>Common implementations use a 32.678 Hz quartz or oscillator.
Low frequency internal RC oscillators (for example 40 kHz) can also be used, but the application 
must calibrate the frequency using a higher accuracy source. With a typical 32 kHz input, the clock resolution 
is about 30 µS and it takes about 17 million years to overflow.</sup>

## The clock counter register

The real-time clock time point register is a 64-bit counter, common on all RV32 and RV64 devices.

To guarantee the steadiness characteristic of the clock, the register is read-only. 
(TODO: define the bit/mechanism that starts it)

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions. 
RV32 devices exposes separate high/low 32-bits registers.

RV64 devices:

- `dcb.rtclock.cnt` 

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [63-0] | `cnt` | ro | RTC timer counter (64-bits). |


RV32 devices:

- `dcb.rtclock.cntl`
- `dcb.rtclock.cnth`

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [31-0] | `cntl` | ro | Low word of RTC timer counter (32-bits). |
| [63-32] | `cnth` | ro | High word of RTC timer counter (32-bits). |

## The clock comparator register

The real-time clock can also be used to trigger periodic interrupts. Low-power devices 
can use the real-time clock to wakeup the entire RISC-V device from implementation 
specific sleep modes.

The comparator register causes a real-time clock interrupt to be posted when the 
counter register 
contains a value greater than or equal to the value in the comparator register.
The interrupt remains posted until it is cleared by writing to the comparator register.

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions. 
RV32 devices exposes separate high/low 32-bits registers.

RV64 devices:

- `dcb.rtclock.cmp` 

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [63-0] | `cmp` | rw | RTC timer comparator (64-bits). |


RV32 devices:

- `dcb.rtclock.cmpl`
- `dcb.rtclock.cmph`

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [31-0] | `cmpl` | ro | Low word of RTC timer comparator (32-bits). |
| [63-32] | `cmph` | ro | High word of RTC timer comparator (32-bits). |

## The clock status and control register

- `dcb.rtclock.ctrl`

TODO: define bits to
- tell is clock is enabled
- enable clock
