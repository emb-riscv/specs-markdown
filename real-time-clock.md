# The Device Real-Time Clock (DRTC)

## Overview

The Device Real-Time Clock is intended to support the implementation of the ISO/IEC 14882.2011 
`system_clock` (20.11.7.1) and `steady_clock` (20.11.7.2) classes. Objects of class 
`system_clock` represent wall clock time from the system-wide realtime clock. Objects of 
class `steady_clock` represent clocks for which values of the time point never decrease as 
physical time advances and for which values of time_point advance at a steady rate 
relative to real time. That is, the clock may not be adjusted.

All harts in a RISC-V device share the same Device Real-Time Clock instance.

## Always on domain

To support full functionality, the Device Real-Time Clock should run even when the 
rest of system is powered down, and so it should be located in a different frequency/voltage 
domain from the processors.

## Clock input

The clock input frequency is fixed, and application or device specific.

To support low-power devices, the clock input should be a low freqency oscilator; the actual 
source is implementation dependent. 

> <sup>Usual implementations use a 32.678 Hz quartz or oscillator.
Internal RC oscillators can also be used, but the application must calibrate the frequency 
using a higher accuracy source.</sup>

## The clock counter register

The clock time point register is a read-only 64-bit counter, common on all RV32, RV64, and 
RV128 devices.

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions. 
RV32 devices exposes separate high/low 32-bits registers.

RV64 devices:

- `dcb.rtclock.counter`: RTC timer count for rdtime (64-bits)

RV32 devices:

- `dcb.rtclock.counterl`: low word of RTC timer count for rdtime (32-bits)
- `dcb.rtclock.counterh`: high word of RTC timer count for rdtimeh (32-bits)









