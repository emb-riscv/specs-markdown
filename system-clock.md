# The Device System Clock (`sysclock`)

## Overview

The **system clock** is intended to support the implementation of the ISO/IEC 14882.2011 
`high_resolution_clock` (§ 20.11.7.3). Objects of class `high_resolution_clock` represent clocks 
with the shortest tick period.

The system clock is also intended as:

- the RTOS tick timer that fires periodically at a programmable rate, for example 1000 Hz, to 
measure time and to drive pre-emptive context switches
- a variable rate alarm or signal timer to handle timeouts and alarms

All harts in a RISC-V device share the same system clock counter, but each hart may have its  
own comparator.

When the device is halted in Debug state, the clock counter is not incremented.

The system clock is inspired by the `mtime`/`mtimecmp` definitions in the RISC-V 
privileged specs, but it differs by counting a higher frequency input, running only when 
the device is powered and not counting during debug.

## Power domain

The system clock is required to run only when the device is powered up, so it can be 
located in the same frequency/voltage domain as the cores.

## Clock input

The system clock source is a reference clock. Software can select whether the reference 
clock is the core clock, the device high frequency reference clock or an implementation 
specific external clock source. If an implementation uses an external clock, it must 
document the relationship between the processor clock and the external reference. 

> <sup>By default, the system clock uses the same source as the core clock, which is 
  a common configuration. 
  For example, with a 100 MHz core clock, the system clock resolution 
  is 10 nS and it takes about 5800 years to overflow.</sup>
  
> <sup>A common RTOS tick frequency is 1000 Hz; in order to accurately achieve this, 
  an input frequency multiple of the tick frequency is required.</sup>
  
> <sup>Low-power devices might need to vary the core frequency by changing implementation 
  specific clock registers (like PLL registers). In this case the system clock software 
  must be notified to use the same input frequency. Alternately, the system clock may 
  be configured to use the high frequency clock reference (like the quartz 
  oscillator), assumed to have a fixed frequency. </sup>

## Memory map

### RV64 devices

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `ctrl` | 32b | rw | 0x00000003 | Control and status register. |
| 0x0008 | `cnt` | 64b | ro | 0x00000000'00000000 | System clock timer counter. |

Part of the Hart Control Block

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `cmp` | 64b | rw | Undefined | System clock timer comparator. |

### RV32 devices

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `ctrl` | 32b | rw | 0x00000003 | Control and status register. |
| 0x0008 | `cntl` | 32b | ro | 0x00000000 | Low word of system clock timer counter. |
| 0x000C | `cnth` | 32b | ro | 0x00000000 | High word of system clock timer counter. |

Part of the Hart Control Block

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `cmpl` | 32b | rw | Undefined | Low word of system clock timer comparator. |
| 0x0004 | `cmph` | 32b | rw | Undefined | High word of system clock timer comparator. |

## The clock control and status register

Controls the system clock timer and provides status data.

By default, the system clock starts disabled; software must enable it during startup.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | `enable` | rw | 0b0 | Indicates the enabled status of the system clock counter: <br> 0 - Counter is disabled (default). <br> 1 - Counter is enabled. |
| [2-1] | `source` | rw | 0b11 | Indicates the clock source: <br> 0b00 - Implementation specific external reference clock. <br> 0b01 - Reserved. <br> 0b10 - High frequency clock reference. <br> 0b11 - Core clock (default). |
| [31-3] |||| Reserved. |

## The clock counter register

The system clock time point register is a 64-bits counter, common on all RV32 and RV64 devices.

To guarantee the steadiness characteristic of the clock, the register is read-only. At reset, the register is cleared to 0.

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions. 
RV32 devices exposes separate high/low 32-bits registers.

## The clock comparator register

In addition to keeping track of time, the system clock can also be used to trigger 
interrupts at specific time points, either for periodic events (like driving 
pre-emption in a RTOS scheduler) or to trigger timeout events.

The comparator register causes a `sysclock_cmp` interrupt to be posted when the 
counter register 
contains a value greater than or equal to the value in the comparator register.
The interrupt remains posted until it is cleared by writing to the comparator register.

The clock comparator register is specific to each hart and is part of the Hart Control Block.

Only hart 0 is required to have a comparator. If any other harts also have comparators, 
the `sysclock_cmp` interrupt is posted only to the local hart. For harts that do not have
a comparator, this register always reads as 0 and writes are ignored.

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions. 
RV32 devices exposes separate high/low 32-bits registers.

## Usage

The memory mapped registers are available via a set of structures, directly available in C/C++.

RV64 devices:

- `sysclock.ctrl`
- `sysclock.cnt` 
- `hcb.sysclockcmp` 

RV32 devices:

- `sysclock.ctrl`
- `sysclock.cntl`
- `sysclock.cnth`
- `hcb.sysclockcmpl`
- `hcb.sysclockcmph`

```c
uint64_t 
riscv_sysclock_read_cnt(void)
{
#if __riscv_xlen == 32
  // Atomic read. The loop is taken once in most cases. Only when the
  // value carries to the high word, two loops are performed.
  while (true)
    {
      uint32_t hi = sysclock.cnth;
      uint32_t lo = sysclock.cntl;
      if (hi == sysclock.cnth)
        {
          return ((uint64_t) hi << 32) | lo;
        }
    }
#else
  return sysclock.cnt;
#endif
}

uint64_t 
riscv_sysclock_read_cmp(void)
{
#if __riscv_xlen == 32
  return ((uint64_t) hcb.sysclockcmph << 32) | hcb.sysclockcmpl;
#else
  return hcb.sysclockcmp;
#endif
}

void 
riscv_sysclock_write_cmp(uint64_t value)
{
#if __riscv_xlen == 32
  // Write low as max; no smaller than old value.
  hcb.sysclockcmpl = (uint32_t) UINT_MAX;
  // Write high; no smaller than old value.
  hcb.sysclockcmph = ((uint32_t) (value >> 32));
  // Write low as new value.
  hcb.sysclockcmpl = ((uint32_t) value);
#else
  hcb.sysclockcmp = value;
#endif
}
```

A typical periodic tick counter:

```c

uint64_c sysclock_cmp;
uint32_t sysclock_increment;

void
sysclock_init(void)
{
  // ...
  sysclock_increment = INPUT_FREQ_HZ/SYSCLOCK_FREQ_HZ;
  sysclock_cmp = riscv_sysclock_read_cnt() + sysclock_increment;
  
  // Ask for an interrupt after one tick interval.
  // Since the comparator is not initialised at reset, it
  // must be written before enabling interrupts.
  riscv_sysclock_write_cmp(sysclock_cmp);
  
  // Assign a priority.
  hic.interrupts[SYSCLOCK_CMP_INT_NUM].prio = SYSCLOCK_CMP_PRIO;
  // Enable.
  hic.interrupts[SYSCLOCK_CMP_INT_NUM].status = INTERRUPTS_SET_ENABLED;
}

void
interrupt_handle_sysclock_cmp(void)
{
  // Increment the clock tick counter and run tick actions.
  sysclock_tick_increment();
  
  // Compute the next time point when the interrupt should come.
  sysclock_cmp += sysclock_increment;
  riscv_sysclock_write_cmp(sysclock_cmp);
}

```
