# Exceptions and Interrupts

Exceptions are unusual **conditions that occur at run time, associated with an
instruction** in the current RISC-V hart.

Interrupts are **events that occur asynchronously outside** any of the RISC-V harts.

> <sup>Other architectures define interrupts as a specific type of exceptions.
  However, for the RISC-V microcontroller profile, exceptions are specific
  for the architecture, and common to all devices, while interrupts are
  mostly specific to an implementation (except a few system interrupts,
  also common to all devices). Thus it looks more natural to define
  two separate vector tables, one for exceptions, to be implemented
  in the architecture software package, and one for interrupts, to be
  implemented in the device software package.</sup>

The mechanism to process exceptions and interrupts (vectored, nested, separate stack)
is one of the main improvements in the RISC-V microcontroller profile over the
privileged profile.

## Exception and interrupt handlers in C/C++

The main feature is the ability to write the exception and interrupt handlers
as plain C/C++ function, that do not need any compiler attributes, or assembly
code.

For this to be possible, there are two requirements:

- both the exception and the interrupt entry code must abide by the ABI requirements
and save the same caller registers as a regular C/C++ call
- a custom return address must be used, such that when the handler returns,
the core will trigger the exception return mechanism, without the need of explicit
assembly `mret` instructions.

## Exceptions

Exceptions trigger a **synchronous transfer of control** to an exception handler
within the current hart.

Some exceptions cannot be disabled, and handlers to process them should always be installed.

Some exceptions are **resumable**, i.e. an execution can continue to the next
instruction (for example the illegal instruction handler can implement a custom
instruction and resume).

The RISC-V privileged specs define the following exceptions, in decreasing priority order:

* Instruction address misaligned
* Instruction access fault
* Illegal instruction
* Breakpoint
* Load address misaligned
* Load access fault
* Store/AMO address misaligned
* Store/AMO access fault
* Environment call from U-mode
* Environment call from M-mode
* Instruction page fault
* Load page fault
* Store/AMO page fault

TODO: rework for microcontrollers; define which one have configurable priorities.

TODO: NMI? routed only to hart 0?

### Exceptions vector table

The exceptions vector table is an array of addresses (xlen size elements) pointing to
interrupt handlers (C/C++ functions).

The address of the exceptions vector table is kept by each hart in (`hcb.excvta`);
it is automatically initialised at startup with
the address provided in the hart startup block and can be later written by software.

## Interrupts

Interrupts are generally **triggered by peripherals** to notify the application of a
given condition or event.

Interrupts trigger the transfer of control to an interrupt handler associated with
a hart.

In the RISC-V microcontroller profile, a hart can have up to **1024** interrupts,
including the system interrupts.

> <sup>This limit was chosen arbitrarily and is considered quite high.</sup>

### Interrupt priorities

Interrupts have **programmable priorities**, defined as small unsigned numbers.

The **priority value 0** is reserved to mean
_'never interrupt'_ or _'disabled'_, and interrupt priorities increase with
the increasing integer value.

Interrupts with the same priority are processed in the order of their index
in the interrupt vector
table, with a higher index meaning a higher priority.

For multi-hart devices, the interrupt wiring to harts is implementation specific;
each interrupt
may be wired to one or several harts; it is the responsibility
of each hart to enable the interrupts it desires to process. For redundant systems,
it is also
possible for multiple harts to process the same interrupt.

### Interrupt priority threshold

Each hart has an associated priority threshold, held in a hart-specific register.

Only interrupts that have a priority strictly greater than the threshold will
cause an interrupt to be sent to the hart.

### Priority bits

The actual number of bits used to store the interrupt priority is implementation
specific, but must
be at least 3 (i.e. at least 8 priority levels).

> <sup>Extra care must be considered when moving code to implementations with fewer
  priority levels, since truncation could lead to priority inversions.
  For example, when moving a program from devices
  with 4-bit priority bits to devices with 3-bit priorities, if the application
  uses priority 9 for IRQ0 and priority 3
  for IRQ1, IRQ0 is expected to have a higher
  priority. But if the MSB bit is removed, IRQ0 will have priority 1 and be
  lower than IRQ1.</sup>

> <sup>It is
  recommended that software handling priorities know about the number of bits
  and use asserts to validate the priority values.</sup>

> <sup>[PA]: the truncation of priority bits should be done at the
 least-significant end, to avoid the kind of
 priority inversion. [ilg] this translates into moving the bits to the other
 end of the word/register, and possibly requiring byte/half-word accesses
 to the NIC.<sup>

### Interrupt preemption and nesting

If an hart is executing an interrupt handler and a higher priority interrupt
occurs, the current interrupt handler is temporarily suspended and the higher
priority interrupt handler is executed to completion, then the initial
interrupt handler is resumed.

Each new interrupt creates a new context on the main stack, and removes it
when the handler returns.

There is no limit for interrupt nesting, assuming the main stack is large enough.

### System interrupts

System interrupts are generated by system peripherals, like `sysclock`, `rtclock`.

TBD

### Interrupts vector table

The interrupts table is an **array of pointers** to interrupt handlers,
implemented as **C/C++ functions**. The number of interrupts per hart is
implementation specific but cannot exceed 1024 elements.

Each hart may have its own table, with handlers for the interrupts it can process.

The address of the array must be programmatically written by each hart to
its `hcb.intvta` register before enabling interrupts, usually during startup.

The first 8 entries are reserved for system interrupts:

* `context_switch` (must have the lowest priority)
* `rtclock_cmp`
* `sysclock_cmp`
* ... 5 more, reserved

## Vector tables relocation

The starting address used by a RISC-V microcontroller is usually
either a flash memory or a ROM device, and the value cannot be changed at run-time.
However, some applications, like bootloaders or applications running in RAM,
start with the vector tables at one
address and later transfer control to the application located at a different
address. For such cases it is useful to be able to modify or define vector tables
at run-time. In order to handle this, the RISC-V microcontrollers support a feature
called Vector Table Relocation.

For this, the `hcb.excvta` and `hcb.intvta` registers can be written at any time from
code running in machine mode.

## Context stack

When exceptions/interrupts are taken, they push a context on the current stack.
The stack pointer must be xlen aligned. For RV32 harts with the D extension,
an additional alignment to 8 may be required.

If the `stackalign` bit in the `ctrl` CSR is set, the stack is always aligned
at 8. Although this is implementation specific, it usually allows faster context
switches.

The RISC-V microcontroller profile uses a full-descending context stack, where:

- When pushing context, the hardware decrements the stack pointer to the end of the
new stack frame before it stores data onto the stack.
- When popping context, the hardware reads the data from the stack frame and then
increments the stack pointer.

The current stack pointer is either `spt` (when in application mode and the
`ctrl.sptena` is set),
or `spm` otherwise (when already in handler mode or `ctrl.sptena` is not set).

In other words, regardless how many nested interrupts occur, there is only one
context pushed onto the thread stack, and all other nested contexts are pushed
onto the main stack. Also
all handlers use the main stack, and do not pollute the thread stack, which
do not need to reserve space for the interrupt handlers.

For the current RISC-V Linux ABI, the stack context is, from hight to low
addresses:

- <-- original `sp` (`spt` or `spm`)
- (optional padding)
- `status` (CSR, the current mode when the exception/interrupt occurred)
- `pc` (the next address to return from the exception/interrupt)
- `x31/t6`
- `x30/t5`
- `x29/t4`
- `x28/t3`
- `x17/a7`
- `x16/a6`
- `x15/a5`
- `x14/a4`
- `x13/a3`
- `x12/a2`
- `x11/a1`
- `x10/a0`
- `x7/t2`
- `x6/t1`
- `x5/t0`
- `x1/ra` <-- new `sp`, possibly align 8

With the new RISC-V EABI proposal, this would be reduced to a more
reasonable context stack:

- <-- original `sp` (`spt` or `spm`)
- (optional padding)
- `status` (CSR, the current mode when the exception/interrupt occurred)
- `pc` (the next address to return from the exception/interrupt)
- `x15/a5`
- `x14/a4`
- `x13/a3`
- `x12/a2`
- `x11/a1`
- `x10/a0`
- `x1/ra` <-- new `sp`, possibly align 8

With floating point support added, the context stack for the current RISC-V
Linux ABI is quite large, which is another good reason why the RISC-V
microcontroller profile should use an optimised Embedded ABI.

- <-- original `sp` (`spt` or `spm`)
- (optional padding)
- `fcsr` (\*) <- for double, it must be aligned to 8
- `f31/ft11` (\*)
- `f30/ft10` (\*)
- `f29/ft9` (\*)
- `f28/ft8` (\*)
- `f17/fa7` (\*)
- `f16/fa6` (\*)
- `f15/fa5` (\*)
- `f14/fa4` (\*)
- `f13/fa3` (\*)
- `f12/fa2` (\*)
- `f11/fa1` (\*)
- `f10/fa0` (\*)
- `f7/ft7` (\*)
- `f6/ft6` (\*)
- `f5/ft5` (\*)
- `f4/ft4` (\*)
- `f3/ft3` (\*)
- `f2/ft2` (\*)
- `f1/ft1` (\*)
- `f0/ft0` (\*)
- `status` (CSR, the current mode when the exception/interrupt occurred)
- `pc` (the next address to return from the exception/interrupt)
- `x31/t6`
- `x30/t5`
- `x29/t4`
- `x28/t3`
- `x17/a7`
- `x16/a6`
- `x15/a5`
- `x14/a4`
- `x13/a3`
- `x12/a2`
- `x11/a1`
- `x10/a0`
- `x7/t2`
- `x6/t1`
- `x5/t0`
- `x1/ra` <-- new `sp`, possibly align 8

(\*) The floating point registers are not saved by devices that do not
implement the
F or D extensions and do not have the `ctrl.fpena` bit set.

To reduce latency, in parallel with saving the registers, the address of the
exception/interrupt handler is fetched from the vector table.

After saving the context stack:

- the `handler` bit in the `status` register is set, to mark the handler-mode
- the `ra` register is loaded with a special HANDLER_RETURN pattern,
defined below
- the `pc` register is loaded with the handler address; this is equivalent
with calling the handler.

When the C/C++ function returns, the return code will load `pc` with the
special HANDLER_RETURN value in `ra`.
This will trigger the exception return mechanism, which will pop the context
from the stack and return from the interrupt/exception.

TODO: define the detailed logic in pseudocode.

## The HANDLER_RETURN pattern

The special HANDLER_RETURN pattern is an 'all-1' for the given xlen with
some bits used to differentiate contexts.
Since the RISC-V microcontroller profile reserves a slice at the very end
of the memory space (0xF...), and this slice has the execute permissions
removed, it does not create any confusion.

This value is generated at exception entrance and is stored in the return
address register (`ra`).

The HANDLER_RETURN pattern bits:

| Bits | Value | Description |
|:-----|:------|-------------|
| [0] | 1 | Reserved. |
| [1] | 0 | Reserved. |
| [2] | - 0: main stack<br>- 1: thread stack | Stack that holds the context to pop. |
| [3] | - 0: short, without FP<br>- 1: long, with FP | Stack frame type. |
| [4] | - 0: Linux<br>- 1: Embedded | ABI |
| [(xlen-1):5] | 1 | Reserved. |

> <sup>The ABI bit is used mainly for compatibility reasons, until the EABI
  will be finalised and implemented by the compiler.</sup>

> <sup>The HANDLER_RETURN pattern does not include a bit defining the
  resulting
  application/handler mode, since it can be restored from the saved
  `status` register. Saving this register is necessary not only for the
  `handler` bit (which might have been added to HANDLER_RETURN), but for the
  `cause` field, which otherwise may be overridden by nested interrupts.</sup>

## The FP lazy stacking mechanism

The large number of floating point registers take a long time to copy
during context push/pop on the stack.

One solution to optimize this is to save them only when needed, by using a
lazy stacking mechanism.

TODO: define the details.

## Tail chaining

When an exception/interrupt takes place while already in handler mode, and the
priority does not require pre-emption, the new exception/interrupt will enter the
pending state. When the hart finishes executing the current handler, it can then
proceed to process the pending exception/interrupt request. Instead of restoring
the registers back from the stack (unstacking) and then pushing them on to the
stack again (stacking), the hart skips the unstacking and stacking steps and
enters the new handler of the pending exception/interrupt as soon as possible.

TODO: define the details.

## Usage

```c
extern "C" {

riscv_startup_block_t
__attribute__((section(".startup_blocks")))
harts_startup_blocks[] = {
  {
    hart_startup,
    hart_stack_pointer,
    hart_global_pointer,
    hart_exception_handlers // <--
  }
};

// The exception vector table address is automatically set during startup.
riscv_exception_handler_t
hart_exception_handlers[] = {
  exception_handle_address_misaligned,
  exception_handle_address_fault,
  exception_handle_illegal_instruction,
  // ...
};

// An example of an exception handler. Plain C function. May return.
void
exception_handle_address_misaligned()
{
  // ...
}

// ...

[[noreturn]] void
hart_startup(void)
{
  // ...
  // Set the interrupt vector table address.
  hcb.intvta = hart_interrupt_handlers;
  // ...
}

riscv_interrupt_handler_t
hart_interrupt_handlers[] = {
  interrupt_handle_context_switch,
  interrupt_handle_rtclock_cmp,
  interrupt_handle_sysclock_cmp,
  // ...
};

// ...

// An example of an interrupt handler. Plain C function.
void
interrupt_handle_syslock_cmp(void)
{
  // ...

  // Simply returns without having to do anything special.
}

} // extern "C"
```
