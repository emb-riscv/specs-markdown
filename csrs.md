# The Control and Status Registers (CSRs)

The RISC-V ISA defines a set of 4096 Control and Status Registers that can be accessed 
via special `csr` instructions with immediate operands identifying the register.

In the RISC-V microcontroller profile only a small number of core system registers 
are accessible via the `csr` instructions, the rest being memory mapped.

## Machine Status Register (`mstatus`)

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [2:0] | | | | Reserved. |
| [3] | `mie` | rw | 0 | Machine Interrupts Enable; 1 if interrupts are enabled. |
| [(xlen-1):4] | | | 0 | Reserved. |

## Machine Exception Program Counter (mepc)

`mepc` is an XLEN-bit read/write register that holds the trap address. The low bit of 
`mepc` (`mepc[0]`) is always zero. 

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | | Reserved. |
| [(xlen-1):1] | `mepc` | rw | Unknown | Exception PC address. |

On implementations that do not support instruction-set 
extensions with 16-bit instruction alignment, the two low bits (`mepc[1:0]`) are always zero.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [1:0] | | | | Reserved. |
| [(xlen-1):2] | `mepc` | rw | Unknown | Exception PC address. |

When an exception is taken, `mepc` is written with the address of the instruction that 
encountered the exception. Otherwise `mepc` is never written by the implementation.

> <sup>In the privileged specs, `mepc` may be explicitly written by software. Here this
it is not needed, execution will resume to the address pushed onto the main stack.</sup>

## Machine Cause Register (mcause)

The `mcause` register is an XLEN-bit read-write register. When an exception or interrupt is
taken `mcause` is written with a code indicating the event that caused the exception or the
interrupt. Otherwise `mcause` is never written by the implementation.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [9:0] | `cause` | rw | Unknown | The cause code. |
| [(xlen-2):10] | | | | Reserved. |
| [(xlen-1)] | `interrupt` | rw | Unknown | 1 if interrupt, 0 if exception. |

## RISC-V microcontroller specific registers

- `hcb.msp`: main stack poiner (xlen)
- `hcb.msplimit`: the lowest address the main stack can descend (xlen)
- `hcb.tsp`: thread stack poiner (xlen)
- `hcb.tsplimit`: the lowest address the thread stack can descend (xlen)

## RISC-V compatibility CSRs

These registers are mandated by the RISC-V Volume I, Chapter 2.8, for the `rdcycle` and `rdinstret` instructions.

RV32/RV64 devices:

- `mhartid`: the current hart ID (32-bits)

Other RISC-V registers from RISC-V Volume II, but not actively used:

- mie: separate interrupts are enabled in the HIC registers
- mtvec: there are two memory mapped registers, `excvta` and `intvta`
- mscratch: not decided if needed
- mepc: probably not needed, traps use the separated stack
- mcause 
- mtval 
- mip 
- misa 


TODO: 

- add MPU registers

