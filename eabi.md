# Embedded ABI

The current RISC-V privileged ABI requires the caller to save the following registers:
`ra`, `t0`, `t1`, `t2`, `a0`, `a1`, `a2`, `a3`, `a4`, `a5`, `a6`, `a7`, `t3`, `t4`,
`t5`, `t6`. This amounts
to 16 registers. If floating point is used, 20 more registers must be saved.

In order to be able to call a C/C++ function from the interrupt handler, all
these registers must be saved when entering interrupts, which impacts the
interrupt latency.

## Proposal

The main goal of the RISC-V Embedded ABI is to balance a high performance for background code with a reduced interrupt latency.

As a secondary goal, if possible, it should remain consistent when applied to the reduced register set used by the RV32E devices.

TODO: This is a very preliminary proposal and must be discussed thoroughly with the compiler maintainers.

TODO: Check if it is ok to use x6 for the stack limit. In Volume I, 2.5, it is mentioned as alternate link register.

### RV32E EABI calling convention

| Register | ABI Name | Description | Caller | Callee |
|:---------|:---------|:------------|--------|-------|
| `x0` | `zero` | Hard-wired zero |  |  |
| `x1` | `ra` | Return address | * |  |
| `x2` | `sp` | Stack pointer |  | * |
| `x3` | `gp` | Global pointer |  |  |
| `x4` | `tp` | Thread pointer |  |  |
| `x5` | `t1/al` | Temporary/alternate link register | * | |
| `x6` | `s3` | Saved register |  | * |
| `x7` | `s4/sl` | Saved register/stack limit |  | * |
|||||
| `x8` | `s0/fp` | Saved register/frame pointer |  | * |
| `x9` | `s1` | Saved register |  | * |
| `x10,x11` | `a0,a1` | Function arguments/return values | * |  |
| `x12` | `a2` | Function arguments | * |  |
| `x13` | `a3` | Function arguments | * |  |
| `x14` | `s2` | Saved register |  | * |
| `x15` | `t0` | Temporary | * | |

> <sup>[AW] For RVC compressibility, the most popular registers should 
  be `x8-x15`. So I suggest renumbering `x8/x9` to be `s0/s1` (as is the 
  case in the POSIX ABI).</sup>
  
> <sup>[AW] Per the ISA spec, `x5` serves as an alternate link register, 
  to drive hardware management of return-address stacks. It’s used by 
  things like the `-msave-restore` option, which reduces code size by 
  using millicode routines to implement prologues/epilogues. For this 
  to work, `x5` needs to be one of the t-registers, as is the case in 
  the POSIX ABI. I suggest either adding one more t-register at `x5`, 
  or moving an existing t-register to `x5`. (The former option is better 
  for code size and performance; the latter option is better for 
  interrupt latency.)</sup>
  
> <sup>[AW] For all the use cases I’ve encountered, two t-registers 
  is sufficient for linkage purposes.</sup>
  
> <sup>[BH] `jal[r]` and `jr` with `x5` are being baked into hardware as 
  function call/return, just as with `x1`, complete with a special 
  return address stack to accelerate the indirect jump for function 
  return. That's *especially* important with millicode register 
  save/restore which will primarily be used on microcontrollers. 
  So `x5` must be a t register. No choice. But maybe don't call it 
  `t0`, if at least one t register is in `x8-x15`.</sup>
  
> <sup>[BH] the lowest numbered registers of each class (s, a t) should 
  fall somewhere inside the C-favoured registers `x8-x15` (if any  
  registers of that class fall in this range).</sup>
  
> <sup>Having the stack limit exposed as a general register
  would save an extra push/pop during RTOS context switches.</sup>

> <sup>[BH] I don't like the stack limit being in a register.
  Much better in a CSR. Harder to corrupt by accident.</sup>
 
More details on the register allocation in the 
[SW Dev email group](https://groups.google.com/a/groups.riscv.org/d/msg/sw-dev/Lp6ucrijap0/ZwVO5Ts-CQAJ).

### RV32I/RV64I EABI calling convention

| Register | ABI Name | Description | Caller | Callee |
|:---------|:---------|:------------|--------|-------|
| `x0` | `zero` | Hard-wired zero |  |  |
| `x1` | `ra` | Return address | * |  |
| `x2` | `sp` | Stack pointer |  | * |
| `x3` | `gp` | Global pointer |  |  |
| `x4` | `tp` | Thread pointer |  |  |
| `x5` | `t1/al` | Temporary/alternate link register | * | |
| `x6` | `s3` | Saved register |  | * |
| `x7` | `s4/sl` | Saved register/stack limit |  | * |
|||||
| `x8` | `s0/fp` | Saved register/frame pointer |  | * |
| `x9` | `s1` | Saved register |  | * |
| `x10,x11` | `a0,a1` | Function arguments/return values | * |  |
| `x12` | `a2` | Function arguments | * |  |
| `x13` | `a3` | Function arguments | * |  |
| `x14` | `s2` | Saved register |  | * |
| `x15` | `t0` | Temporary | * | |
|||||
| `x16–x31` | `s5-s20` | Saved registers |  | * |
|||||
| `f0–f1` | `fa0-fa1` | FP arguments/return values | * |  |
| `f2–f7` | `fa2-fa7` | FP arguments | * |  |
| `f8–f15` | `ft0-ft7` | FP temporaries | * |  |
| `f16–f31` | `fs0-fs15` | FP saved registers |  | * |

> <sup>To simplify the context push/pop code,
  the floating point registers were reordered, to group
  all the caller register in one half of the set and the callee
  saved registers in the other half.</sup>

### Sizes of variables

- `long double` - 64 bits.

TODO: add all other

## References

## RISC-V POSIX ABI

Currently defined in Chapter 20, RISC-V Assembly Programmer’s Handbook, of the "The RISC-V Instruction Set Manual Volume I: User-Level ISA, Document Version 2.2".

| Register | ABI Name | Description | Caller | Callee |
|:---------|:---------|:------------|--------|-------|
| `x0` | `zero` | Hard-wired zero |  |  |
| `x1` | `ra` | Return address | * |  |
| `x2` | `sp` | Stack pointer |  | * |
| `x3` | `gp` | Global pointer |  |  |
| `x4` | `tp` | Thread pointer |  |  |
| `x5` | `t0` | Temporary/alternate link register | * |  |
| `x6–x7` | `t1-t2` | Temporaries | * |  |
| `x8` | `s0/fp` | Saved register/frame pointer |  | * |
| `x9` | `s1` | Saved register |  | * |
| `x10–x11` | `a0-a1` | Function arguments/return values | * |  |
| `x12–x17` | `a2-a7` | Function arguments | * |  |
| `x18–x27` | `s2-s11` | Saved registers |  | * |
| `x28–x31` | `t3-t6` | Temporaries | * |  |
|||||
| `f0–f7` | `ft0-ft7` | FP temporaries | * |  |
| `f8–f9` | `fs0-fs1` | FP saved registers |  | * |
| `f10–f11` | `fa0-fa1` | FP arguments/return values | * |  |
| `f12–f17` | `fa2-fa7` | FP arguments | * |  |
| `f18–f27` | `fs2-fs11` | FP saved registers |  | * |
| `f28–f31` | `ft8-ft11` | FP temporaries | * |  |

## RISC-V RV32E ABI

Currently defined in the [RISC-V ELF psABI](https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md#-rv32e-calling-convention).

| Register | ABI Name | Description | Caller | Callee |
|:---------|:---------|:------------|--------|-------|
| `x0` | `zero` | Hard-wired zero |  |  |
| `x1` | `ra` | Return address | * |  |
| `x2` | `sp` | Stack pointer |  | * |
| `x3` | `gp` | Global pointer |  |  |
| `x4` | `tp` | Thread pointer |  |  |
| `x5` | `t0` | Temporary/alternate link register | * |  |
| `x6–x7` | `t1-t2` | Temporaries | * |  |
| `x8` | `s0/fp` | Saved register/frame pointer |  | * |
| `x9` | `s1` | Saved register |  | * |
| `x10–x11` | `a0-a1` | Function arguments/return values | * |  |
| `x12–x15` | `a2-a5` | Function arguments | * |  |

## Links

- [Application Binary Interface for
the ARM® Architecture](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0036b/IHI0036B_bsabi.pdf)
- [Procedure Call Standard for the ARM® Architecture](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042f/IHI0042F_aapcs.pdf)
- [RISC-V ELF psABI](https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md)

