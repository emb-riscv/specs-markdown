# Embedded ABI

The current RISC-V ABI requires the caller to save the following registers: 
ra, t0, t1, t2, a0, a1, a2, a3, a4, a5, a6, a7, t3, t4, t5, t6. This amounts 
to 16 registers. If floating point is used, 20 more registers must be saved.

In order to be able to call a C/C++ function from the interrupt handler, all 
these registers must be saved when entering interrupts, which impact the
interrupt latency.

## Proposal

The main goal of the RISC-V Embedded ABI is to balance a high performance for background code with a reduced interrupt latency.

As a secondary goal, if possible, it must remain consistent when applied to the reduced registry set used by the RV32E devices.

TODO: This is a very preliminary proposal and must be discussed thouroghly with the compiler maintainers.

### RV32I/RV64I

| Register | ABI Name | Description | Caller | Calee |
|:---------|:---------|:------------|--------|-------|
| x0 | zero | Hard-wired zero |  |  |
| x1 | ra | Return address | * |  |
| x2 | sp | Stack pointer |  | * |
| x3 | gp | Global pointer |  |  |
| x4 | tp | Thread pointer |  |  |
| x5 | **sl** | **Stack limit** |  | * |
| x6–x7 | t1,t2 | Temporaries | | * |
| x8 | s0/fp | Saved register/frame pointer |  | * |
| x9 | s1 | Saved register |  | * |
| x10,x11 | a0,a1 | Function arguments/return values | * |  |
| x12,x13 | a2,a3 | Function arguments | * |  |
| x14,x15 | a4,a5 | Function arguments | * |  |
|||||
| x16–x17 | ?? | Temporaries? |  | * |
| x18–x27 | s2-s11 | Saved registers |  | * |
| x28–x31 | t3-t6 | Temporaries |  | * |
|||||
| f0–f7 | ft0-ft7 | FP temporaries | * |  |
| f8–f9 | fs0-fs1 | FP saved registers |  | * |
| f10–f11 | fa0-fa1 | FP arguments/return values | * |  |
| f12–f17 | fa2-fa7 | FP arguments | * |  |
| f18–f27 | fs2-fs11 | FP saved registers |  | * |
| f28–f31 | ft8-ft11 | FP temporaries | * |  |

### RV32E

| Register | ABI Name | Description | Caller | Calee |
|:---------|:---------|:------------|--------|-------|
| x0 | zero | Hard-wired zero |  |  |
| x1 | ra | Return address | * |  |
| x2 | sp | Stack pointer |  | * |
| x3 | gp | Global pointer |  |  |
| x4 | tp | Thread pointer |  |  |
| x5 | **sl** | **Stack limit** |  | * |
| x6–x7 | t1,t2 | Temporaries |  | * |
| x8 | s0/fp | Saved register/frame pointer |  | * |
| x9 | s1 | Saved register |  | * |
| x10,x11 | a0,a1 | Function arguments/return values | * |  |
| x12,x13 | a2,a3 | Function arguments | * |  |
| x14,x15 | t3,t4 | Temporary | | * |

## References

## RISC-V Linux ABI

| Register | ABI Name | Description | Caller | Calee |
|:---------|:---------|:------------|--------|-------|
| x0 | zero | Hard-wired zero |  |  |
| x1 | ra | Return address | * |  |
| x2 | sp | Stack pointer |  | * |
| x3 | gp | Global pointer |  |  |
| x4 | tp | Thread pointer |  |  |
| x5 | t0 | Temporary/alternate link register | * |  |
| x6–x7 | t1-t2 | Temporaries | * |  |
| x8 | s0/fp | Saved register/frame pointer |  | * |
| x9 | s1 | Saved register |  | * |
| x10–x11 | a0-a1 | Function arguments/return values | * |  |
| x12–x17 | a2-a7 | Function arguments | * |  |
| x18–x27 | s2-s11 | Saved registers |  | * |
| x28–x31 | t3-t6 | Temporaries | * |  |
|||||
| f0–f7 | ft0-ft7 | FP temporaries | * |  |
| f8–f9 | fs0-fs1 | FP saved registers |  | * |
| f10–f11 | fa0-fa1 | FP arguments/return values | * |  |
| f12–f17 | fa2-fa7 | FP arguments | * |  |
| f18–f27 | fs2-fs11 | FP saved registers |  | * |
| f28–f31 | ft8-ft11 | FP temporaries | * |  |

## Links

- [Application Binary Interface for
the ARM® Architecture](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0036b/IHI0036B_bsabi.pdf)
- [Procedure Call Standard for the ARM® Architecture](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042f/IHI0042F_aapcs.pdf)
