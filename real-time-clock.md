# The Device Real-Time Clock (DRTC)

## Overview

The **Device Real-Time Clock** is intended to support the implementation of the ISO/IEC 14882.2011
`system_clock` (§ 20.11.7.1) and `steady_clock` (§ 20.11.7.2) classes. Objects of class
`system_clock` represent wall clock time from the system-wide real-time clock. Objects of
class `steady_clock` represent clocks for which values of the time point never decrease as
physical time advances and for which values of time_point advance at a steady rate
relative to real time. That is, the clock may not be adjusted.

All harts in a RISC-V device share the same Device Real-Time Clock counter, but each hart may
have its own comparator.

Even when the device is halted in Debug state, the clock counter continues to be incremented.

The real-time clock is inspired by the `mtime`/`mtimecmp` definitions in the RISC-V privileged specs,
but it differs by having a control register and not being intended to drive the scheduler clock.

## Power domain

To support full functionality, the real-time clock should run even when the
rest of system is powered down, so it must be located in a different frequency/voltage
domain from the cores.

## Clock input

The real-time clock input frequency is fixed to a device or application specific value.

To support low-power devices, the real-time clock input should be a low frequency oscillator; the actual
source is implementation specific.

> <sup>Common implementations use a 32.678 Hz quartz or oscillator.
  Low frequency internal RC oscillators (for example 40 kHz) can also be used, but the application
  must calibrate the frequency using a higher accuracy source. With a typical 32 kHz input,
  the clock resolution
  is about 30 µS and it takes about 17 million years to overflow. </sup>

> <sup>The real-time clock is usually not suitable to drive the RTOS tick timer, since either
  it is not accurate enough, or its frequency does not allow the common 1000 Hz scheduler rate;
  use the system clock instead.</sup>

## Memory map

RV64 devices

| Offset | Name | Width | Type | Reset | Description |
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `ctrl` | 32b | rw | 0x00000003 | Control and status register. |
| 0x0008 | `cnt` | 64b | ro | Undefined | RTC timer counter. |
| 0x0010 | `cmp` | 64b | rw | Undefined | RTC comparator. |

RV32 devices

| Offset | Name | Width | Type | Reset | Description |
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `ctrl` | 32b | rw | 0x00000003 | Control and status register. |
| 0x0008 | `cntl` | 32b | ro | Undefined | Low word of RTC counter. |
| 0x000C | `cnth` | 32b | ro | Undefined | High word of RTC counter. |
| 0x0010 | `cmpl` | 32b | rw | Undefined | Low word of RTC comparator. |
| 0x0014 | `cmph` | 32b | rw | Undefined | High word of RTC comparator. |

TODO: define the mechanism to clear the counter. at each enable?

## The clock control and status register

Controls the RTC and provides status data.

By default, the RTC starts disabled; software must enable it during startup.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | `enable` | rw | 0 | Indicates the enabled status of the RTC counter: <br> 0 - Counter is disabled (default). <br> 1 - Counter is enabled. |
| [2-1] | `source` | rw | 0b11 | Indicates the clock source: <br> 0b00 - Implementation specific external reference clock. <br> 0b01 - Reserved. <br> 0b10 - Factory-trimmed on-chip oscillator. <br> 0b11 - External crystal oscillator (default). |
| [31-3] |||| Reserved. |


## The clock counter register

The real-time clock time point register is a 64-bit counter, common on all RV32 and RV64 devices.

To guarantee the steadiness characteristic of the clock, the register is read-only.

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions.
RV32 devices exposes separate high/low 32-bits registers.

## The clock comparator register

In addition to keeping track of time, the real-time clock can also be used to
trigger periodic interrupts. Low-power devices
can use the real-time clock to wakeup the entire RISC-V device from implementation
specific sleep modes.

The comparator register causes a `rtclock_cmp` interrupt to be posted when the
counter register
contains a value greater than or equal to the value in the comparator register.
The interrupt remains posted until it is cleared by writing to the comparator register.

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions.
RV32 devices exposes separate high/low 32-bits registers.

## Usage

The memory mapped registers are available via a set of structures, directly available in C/C++.

RV64 devices:

- `rtclock.ctrl`
- `rtclock.cnt`
- `hcb.rtclockcmp`

RV32 devices:

- `rtclock.ctrl`
- `rtclock.cntl`
- `rtclock.cnth`
- `hcb.rtclockcmpl`
- `hcb.rtclockcmph`

```c
uint64_t
riscv_rtclock_read_cnt(void)
{
#if __riscv_xlen == 32
  // Atomic read. The loop is taken once in most cases. Only when the
  // value carries to the high word, two loops are performed.
  while (true)
    {
      uint32_t hi = rtclock.cnth;
      uint32_t lo = rtclock.cntl;
      if (hi == rtclock.cnth)
        {
          return ((uint64_t) hi << 32) | lo;
        }
    }
#else
  return rtclock.cnt;
#endif
}

uint64_t
riscv_rtclock_read_cmp(void)
{
#if __riscv_xlen == 32
  return ((uint64_t) hcb.rtclockcmph << 32) | hcb.rtclockcmpl;
#else
  return dcb.rtclock.cmp;
#endif
}

void
riscv_rtclock_write_cmp(uint64_t value)
{
#if __riscv_xlen == 32
  // Write low as max; no smaller than old value.
  hcb.rtclockcmpl = (uint32_t) UINT_MAX;
  // Write high; no smaller than old value.
  hcb.rtclockcmph = ((uint32_t) (value >> 32));
  // Write low as new value.
  hcb.rtclockcmpl = ((uint32_t) value);
#else
  hcb.rtclockcmp = value;
#endif
}
```
