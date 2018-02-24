# The Control and Status Registers (CSRs)

The RISC-V ISA defines a set of 4096 Control and Status Registers that can be accessed 
via special `csr` instructions with immediate operands identifying the register.

For performance reasons, in the RISC-V microcontroller profile only a small number 
of core system registers are required to be CSRs; the rest are available in the memory 
mapped system area.

Unless otherwise mentioned, access to these CSRs is limited to machine mode.

## Hart ID Register (`hartid`)

The `hartid` CSR is an XLEN-bit read-only register containing the integer ID of the 
hardware thread running the code. This register must be readable in any implementation. 
In single-hart devices, it always reads 0. In multi-hart devices, the hart IDs might 
not necessarily be numbered contiguously
(althoug it is preferable), but at least one hart must have a hart ID of zero.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [N:0] | `hartid` | ro | | The integer ID of the hart. |
| [(xlen-1):(N+1)] | | | | Reserved. |

N is the number of bits required to store the maximum hart ID and is implementation specific.

This CSR is identical to `mhartid` in the RISC-V privileged profile.

## Interrupt Enable (`iena`)

The `iena` CSR is an XLEN-bit read/write register that controls whether the interrupt 
are enabled or not. 

This register has a single bit on purpose. Access to the interrupt enable bit must be quite 
fast since masking all interrupts is one of the methods used to implement critical sections.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | `iena` | rw | 0 | Interrupts Enable; 1 if interrupts are enabled. |
| [(xlen-1):1] | | | | Reserved. |

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Interrupt Priority Threshold (`iprioth`)

The `iprioth` CSR is an XLEN-bit read/write register that holds the interrupts threshold. 
Only interrupts requests that have a priority strictly greater than the threshold will cause 
an interrupt to become active. The threshold register must always be able to hold the value zero, 
in which case, no interrupts are masked. The threshold register must also be able to hold 
the maximum priority level, in which case all interrupts are masked. 

It is a CSR because access to the interrupt threashold register must be quite fast since 
handling the interrupts threshold is one of the methods used to implement critical sections.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [N:0] | `iprioth` | rw | 0 | The interrupt priority threshold. |
| [(xlen-1):(N+1)] | | | | Reserved. |

N is the number of bits required to store the maximum priority level, and is implementation 
specific. It must match the number of bits used by the `prio` register in the interrupt 
controller.

All reserved bits read back as 0. To find out N at runtime, an application can write an 
all-1 pattern and read back the register.

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Interrupt Priority Threshold Increase (`ipriothinc`)

The `ipriothinc` CSR behaves like an XLEN-bit read/write register, but in fact uses the
same register as `iprioth`. The difference is that writes to this CSR are effective only 
if the new value is higher than the current value, in other words it guarantees that the
interrupt threshold is not decreased.

It is a CSR because access to the interrupt threashold register must be quite fast since 
handling the interrupts threshold is one of the methods used to implement critical sections.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [N:0] | `ipriothinc` | rw | 0 | The interrupt priority threshold. |
| [(xlen-1):(N+1)] | | | | Reserved. |

N is the number of bits required to store the maximum priority level, and is implementation 
specific.

This CSR is specific to the RISC-V microcontroller profile.

TODO: possibly find a better name. how about `ipriothup`?
TODO: allocate a number for it.

## Main Stack Pointer (`msp`)

The `msp` CSR is an XLEN-bit read-write register that holds the main stack pointer. 
It is always the default stack pointer after reset. Interrupts and exceptions always 
use this stack to store the exception frame.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | 0 | Reserved. |
| [(xlen-1):1] | `msp` | rw | startup | The main stack pointer. |

This CSR is specific to the RISC-V microcontroller profile.

TODO: check if the stack has more strict alignment requirements.

TODO: allocate a number for it.

## Main Stack Pointer Limit (`msplimit`)

The `msplimit` CSR is an XLEN-bit read-write register that holds the lowest address 
the main stack can descend.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | 0 | Reserved. |
| [(xlen-1):1] | `msplimit` | rw | startup | The main stack lower limit. |

If an operation using the main stack pointer attempts to write to an address below 
the limit, an exception is triggered and the operation is not performed.

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Thread Stack Pointer (`tsp`)

The `tsp` CSR is an XLEN-bit read-write register that holds the stack pointer used 
by the application current thread.

It is a CSR because access to the stack pointer may occur in context switching 
routines and needs to be fast.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | 0 | Reserved. |
| [(xlen-1):1] | `tsp` | rw | 0 | The thread stack pointer. |

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Thread Stack Pointer Limit (`tsplimit`)

The `tsplimit` CSR is an XLEN-bit read-write register that holds the lowest address 
the thread stack can descend.

It is a CSR because access to the stack pointer limit may occur in context switching 
routines and needs to be fast.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | 0 | Reserved. |
| [(xlen-1):1] | `mtsplimit` | rw | 0 | The thread stack lower limit. |

If an operation using the thread stack pointer attempts to write to an address below 
the limit, an exception is triggered and the operation is not performed.

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Exception Program Counter (`epc`)

The `epc` CSR is an XLEN-bit read/write register that holds the trap address. The low bit of 
`epc` (`epc[0]`) is always zero. 

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | 0 | Reserved. |
| [(xlen-1):1] | `epc` | rw | Unknown | Exception PC address. |

On implementations that do not support instruction-set 
extensions with 16-bit instruction alignment, the two low bits (`epc[1:0]`) are always zero.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [1:0] | | | 0 | Reserved. |
| [(xlen-1):2] | `epc` | rw | Unknown | Exception PC address. |

When an exception is taken, `epc` is written with the address of the instruction that 
encountered the exception. Otherwise `mepc` is never written by the implementation.

This CSR is similar to `mepc` from the RISC-V privileged profile.

> <sup>In the privileged specs, `mepc` may be explicitly written by software. Here this
it is not needed, execution will resume to the address pushed onto the main stack.</sup>

## Exception Cause Register (`ecause`)

The `ecause` CSR is an XLEN-bit read-write register. When an exception or interrupt is
taken `ecause` is written with a code indicating the event that caused the exception or the
interrupt. Otherwise `ecause` is never written by the implementation.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [9:0] | `cause` | rw | Unknown | The exception or interrupt cause code. |
| [(xlen-1):10] | | | | Reserved. |

This CSR is inspired from the RISC-V privileged profile, but does not use the high bit
to differentiate between exceptions and interrupts, because this is not needed
when separate vectors are called.

## Machine Trap Value Register (`mtval`)

The `mtval` CSR is an XLEN-bit read-write register. When an exception is taken,
`mtval` is written with exception-specific information to assist software in handling the 
exception. Otherwise, `mtval` is never written by the implementation, though it may be 
explicitly written by software.

This CSR is identical with `mtval` from the RISC-V privileged profile.

## RISC-V compatibility CSRs

The RISC-V Volume I, Chapter 2.8, mentions two mandatory instructions, `rdcycle` and 
`rdinstret`; to implement them, two CSRs are required:

- cycle: available as `hcb.cyclecnt`
- instret: available as `hcb.instcnt`

The RISC-V Volume II, mentions other CSRs, but it is not clear which one are mandatory, 
if any:

- mstatus: not needed, there is only one bit needed, mie, which is now n the `iena` CSR.
- mie: not needed, interrupts are enabled in the HIC registers
- mip: not needed, interrupt pending bits are in the HIC registers
- mtvec: not needed, there are two memory mapped registers, `excvta` and `intvta`
- misa: can be safely migrated to memory mapped

- mscratch: not decided if needed

TODO: 

- add MPU registers

## Usage

### Interrupt critical sections

There are two ways to implement critical section: if the application does not need to keep any fast interrupts enabled, it can fully disable interrupts; 

```c
function f()
{
  // ...
  {
    // Interrupts critical section.
    xlenreg_t status = riscv_csr_write_iena(0);
    // ...
    riscv_csr_write_iena(status);
  }
  // ...
}
```

Otherwise it can raise the interrupt threashold to a limit below the fast interrupts priority.

```c
function f()
{
  // ...
  {
    // Interrupts critical section.
    xlenreg_t status = riscv_csr_write_ipriothinc(7);
    // ...
    riscv_csr_write_iprioth(status);
  }
  // ...
}
```
