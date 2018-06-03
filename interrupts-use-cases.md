# Appendix B: Interrupts use cases

Regardless how the interrupts are implemented, any architecture design should 
be checked how well the common use cases are accommodated.

## Peripherals vs scheduler interrupts

By design, old microcontroller architectures expected interrupts to be 
triggered only occasionally by peripherals.

Since Cortex-M, interrupts were extensively enhanced with features to
support the implementation of RTOSes, greatly simplifying context switches
and preemption. 

### Peripheral interrupts

Although there were opinions that peripheral interrupts should be as simple
as possible to be fully inlined, a well structured application may use
drivers from a separate library/package, so the typical use case is to
have the interrupt handlers in files specific to the application
and call the driver interrupt service routine via a plain C/C++ call.

The traditional approach is to have the interrupt handler annotated 
with the `interrupt` attribute, which generates a fully functional 
interrupt handler, including preserving registers and returning
from interrupt.

```c
# include "driver-xyz.h"

void __attribute__((interrupt))
interrupt_handle_xyz(void)
{
  driver_xyz_interrupt_service_routine();
}
```

The problem with this approach is that on a RISC-V device, 
with the current POSIX ABI, the number of
registers to be saved by the caller is large, 
and the generated
code, with `-march=rv64gc -mabi=lp64d` looks like:

```
.option nopic 
.text 
.align 1 
.globl interrupt_handle_xyz 
.type interrupt_handle_xyz, @function 
interrupt_handle_xyz: 
addi sp,sp,-288 
sd ra,280(sp) 
sd t0,272(sp) 
sd t1,264(sp) 
sd t2,256(sp) 
sd a0,248(sp) 
sd a1,240(sp) 
sd a2,232(sp) 
sd a3,224(sp) 
sd a4,216(sp) 
sd a5,208(sp) 
sd a6,200(sp) 
sd a7,192(sp) 
sd t3,184(sp) 
sd t4,176(sp) 
sd t5,168(sp) 
sd t6,160(sp) 
fsd ft0,152(sp) 
fsd ft1,144(sp) 
fsd ft2,136(sp) 
fsd ft3,128(sp) 
fsd ft4,120(sp) 
fsd ft5,112(sp) 
fsd ft6,104(sp) 
fsd ft7,96(sp) 
fsd fa0,88(sp) 
fsd fa1,80(sp) 
fsd fa2,72(sp) 
fsd fa3,64(sp) 
fsd fa4,56(sp) 
fsd fa5,48(sp) 
fsd fa6,40(sp) 
fsd fa7,32(sp) 
fsd ft8,24(sp) 
fsd ft9,16(sp) 
fsd ft10,8(sp) 
fsd ft11,0(sp) 
call driver_xyz_interrupt_service_routine 
ld ra,280(sp) 
ld t0,272(sp) 
ld t1,264(sp) 
ld t2,256(sp) 
ld a0,248(sp) 
ld a1,240(sp) 
ld a2,232(sp) 
ld a3,224(sp) 
ld a4,216(sp) 
ld a5,208(sp) 
ld a6,200(sp) 
ld a7,192(sp) 
ld t3,184(sp) 
ld t4,176(sp) 
ld t5,168(sp) 
ld t6,160(sp) 
fld ft0,152(sp) 
fld ft1,144(sp) 
fld ft2,136(sp) 
fld ft3,128(sp) 
fld ft4,120(sp) 
fld ft5,112(sp) 
fld ft6,104(sp) 
fld ft7,96(sp) 
fld fa0,88(sp) 
fld fa1,80(sp) 
fld fa2,72(sp) 
fld fa3,64(sp) 
fld fa4,56(sp) 
fld fa5,48(sp) 
fld fa6,40(sp) 
fld fa7,32(sp) 
fld ft8,24(sp) 
fld ft9,16(sp) 
fld ft10,8(sp) 
fld ft11,0(sp) 
addi sp,sp,288 
mret 
.size interrupt_handle_xyz, .-interrupt_handle_xyz 
```

Simpler devices, without hardware FP, have slightly shorter code,
but still lots of registers (`-march=rv32i -mabi=ilp32`):

```
.option nopic 
.text 
.align 2 
.globl interrupt_handle_xyz 
.type interrupt_handle_xyz, @function 
interrupt_handle_xyz: 
addi sp,sp,-64 
sw ra,60(sp) 
sw t0,56(sp) 
sw t1,52(sp) 
sw t2,48(sp) 
sw a0,44(sp) 
sw a1,40(sp) 
sw a2,36(sp) 
sw a3,32(sp) 
sw a4,28(sp) 
sw a5,24(sp) 
sw a6,20(sp) 
sw a7,16(sp) 
sw t3,12(sp) 
sw t4,8(sp) 
sw t5,4(sp) 
sw t6,0(sp) 
call driver_xyz_interrupt_service_routine 
lw ra,60(sp) 
lw t0,56(sp) 
lw t1,52(sp) 
lw t2,48(sp) 
lw a0,44(sp) 
lw a1,40(sp) 
lw a2,36(sp) 
lw a3,32(sp) 
lw a4,28(sp) 
lw a5,24(sp) 
lw a6,20(sp) 
lw a7,16(sp) 
lw t3,12(sp) 
lw t4,8(sp) 
lw t5,4(sp) 
lw t6,0(sp) 
addi sp,sp,64 
mret 
.size interrupt_handle_xyz, .-interrupt_handle_xyz 
```

On the other hand, modern designs use plain C functions as interrupt 
handlers, and in this case the generated code looks definitely
better:

```
.option nopic 
.text 
.align 2 
.globl interrupt_handle_xyz 
.type interrupt_handle_xyz, @function 
interrupt_handle_xyz:  
tail driver_xyz_interrupt_service_routine 
.size interrupt_handle_xyz, .-interrupt_handle_xyz 
```

For this style of handlers to work, it is still necessary to save/restore
the ABI caller registers outside the handler; this can be done either in
hardware, or, for cheap devices, in software.

### Context switches

In a multi-threaded environment, a context switches is generally 
a sequence of operations performing the following steps:

- interrupt the current thread
- save the state of the current thread in the thread control block (TCB)
- select the next thread to run
- restore the state of the new thread from the selected TCB
- resume execution in the context of the new thread

#### Cooperative vs preemptive

In a cooperative environment, threads deliberately pass control to other
threads either by directly issuing an `yield()` call, or indirectly 
by calling a system function that internally yields.

In a cooperative environment, user interrupt handlers are regular handlers, 
they interrupt the current running code (thread or interrupt), 
perform some operations, and return in exactly the same context.

Preemptive environments improve response time by extending some of
the interrupt handlers with code that also performs context switches,
such that the interrupt occurs in the context of one thread but 
returns in the context of another thread.

Traditional interrupt handlers need to be changed from the simple 
implementation that calls the peripheral ISR:

```c
void __attribute__((interrupt))
interrupt_handle_xyz(void)
{
  driver_xyz_interrupt_service_routine();
}
```

... to something like this:

```c
stack_elem_t* 
static inline __attribute__((naked, always_inline))
save_context(void)
{
  // Assembly code to push all registers onto the thread stack
  // ...
  return sp;
}

void 
static inline __attribute__((naked, always_inline))
restore_context(stack_elem_t* sp)
{
  // Assembly code to pop all registers from the thread stack
  // ...
}


void __attribute__((naked))
interrupt_handle_xxx(void)
{
  stack_elem_t* sp = save_context(); // Push all registers onto the thread stack
  
  driver_xyz_interrupt_service_routine();
  
  if (must_switch_context) {
    sp = scheduler_select_next_thread(sp);
  }
  restore_context(sp); // Pop all registers from the thread stack
  return_from_interrupt();
}
```

The complexity vary from RTOS to RTOS, and in real life it must also include
some critical sections, but the general framework is highly similar to the
above; it requires significant changes in the user code and it is not simple.

#### Dedicated context switch interrupt

In modern RTOS friendly architectures, the context switch is delegated
to a single dedicated interrupt, implemented in the system part, such
that all user interrupt handlers no longer need to worry about this and
can be written directly in C/C++:

```c
void
interrupt_handle_xyz(void)
{
  driver_xyz_interrupt_service_routine();
}
```

... while the context switch is performed by an interrupt handler like:

```c
void
__attribute__((naked))
interrupt_handle_context_switch(void)
{
  stack_elem_t* sp = save_context(); // Push all registers onto the thread stack
  
  sp = scheduler_select_next_thread(sp);
  
  restore_context(sp); // Pop all registers from the stack
}
```

For this to work, the context switch interrupt must be guaranteed to have the
lowest priority, such that it is executed after all other interrupts are 
completed and the hart must return to thread state.

#### Triggering a context switch

With such a dedicated interrupt, triggering a context switch is as simple 
as pending a software interrupt:

```c
hcb.interrupts[CONTEXT_SWITCH_INTERRUPT_NUMBER].status = INTERRUPTS_SET_PENDING;
```

Pending a context switch interrupt can be performed either in other interrupt
handlers (and in this case the switch occurs after all interrupts are 
completed), or in thread mode, in the `yield()` function, and in this case
the switch is performed as soon as interrupts are enabled.

## Use cases

Once defined the mechanism to switch contexts via a dedicated interrupt, 
it is easy to imagine that, with a preemptive scheduler,
most peripheral interrupts can trigger context switches,
so it becomes clear that both peripheral and context switch interrupts 
should be given equal attention in the design.

The most general case is when a peripheral interrupt occurs, while it is 
processed other interrupts with lower or equal priorities occur too and
wait their turn, and are processed back-to-back,
and one of those interrupts requests a context switch,
so the last interrupt in the chain is the context switch interrupt.

From simple to complex, the use cases are:

### Single peripheral interrupt, no context switch

This is the simplest case, when the driver processes the peripheral
data, but does not need to inform the associated thread of the
change, so it does not request a context switch, and after the 
interrupt completed, execution returns to the same thread. 

### Single peripheral interrupt with context switch

If the driver decides to inform the thread that new data is available,
for example by raising a semaphore, or pushing data onto a queue, it
must pend the context switch interrupt, which will be executed
back-to-back with the peripheral interrupt.

### Multiple interrupts with context switch

If, during the execution of the peripheral interrupt, other
interrupts with lower or equal priority occur, they do not
preempt the current interrupt, but are remembered and when
interrupt completes are executed in sequence, back-to-back,
including the context switch interrupt, if requested.

## Tail chaining

Given the use cases presented, with virtually all
peripheral interrupts requesting context switches,
it results that it is highly likely
to have at least two back-to-back interrupts.

Old architectures that use interrupt handlers annotated 
with the `interrupt` attribute, simply call the handlers
in sequence, and each handler saves and restores all registers.

For back-to-back interrupts, the registers restores by
the first interrupt have exactly the same values as those
saved by the second interrupt, so the long list of
register operations is practically useless, but the
compiler does not know this, so the code is not efficient.

For the current RISC-V POSIX ABI, the behaviour is:

- process the top priority interrupt
  - enter annotated handler 
  - **save 16 general registers and 20 FP registers**
  - call the C/C++ functions and return
  - **restore 16 general registers and 20 FP registers**
  - exit annotated handler
- possibly process other interrupts with lower or similar 
priority that occur while in interrupt mode, each of them doing (**N times**)
  - enter annotated handler 
  - **save 16 general registers and 20 FP registers**
  - call the C/C++ functions and return
  - **restore 16 general registers and 20 FP registers**
  - exit annotated handler
- process the context_switch interrupt (lowest possible priority)
  - enter naked handler
  - **save 32 general registers and 32 FP registers**
  - save the SP in the current thread control block
  - select the next thread to run
  - load SP from the new thread control block
  - **restore 32 general registers and 32 FP registers**
  - exit naked handler
- return from interrupt in the context of the new thread

In modern designs, which use plain C interrupt handlers,
the registers are saved before entering the first handler
and restored after the first handler,
so the behaviour is significantly more efficient:

- reserve space for the FP registers, but do not save them
- **save** the ABI caller registers
- call the handler for top priority interrupt
- possibly call other handlers for interrupts with lower or similar 
priorities, that occur while in interrupt mode
- call the `context_switch` handler (lowest possible priority)
  - save the rest of the general registers (ABI callee)
  - save the SP in the current thread control block
  - select the next thread to run
  - load the SP from the new thread control block
  - restore the rest of the general registers (ABI callee)
  - return from the handler
- **restore** the ABI caller registers
- return from interrupt in the context of the new thread

## Lazy FP stacking

For devices with hardware FP units, the large number of FP
registers may severely impact the interrupt latency.

Old architectures that use interrupt handlers annotated 
with the `interrupt` attribute, should always save and 
restore all the FP registers, as seen in the example.

In modern designs, which use plain C interrupt handlers,
and the registers are saved before entering the handlers,
it is possible to use a more efficient mechanism, which
only reserves the space onto the thread stack, but does
the actual save only when the first FP instructions is 
executed.

Since most interrupt handlers do not use FP instructions,
the saving/restoring of the FP registers is skipped 
entirely, and the interrupt latency is not affected.

> <sup>More details on this mechanism to be added as a separate page. 
The design should also consider the ABI callee registers,
handled during context switches.</sup>

## Conclusions

When designing a new architecture, the focus should be
to optimise the most common use case, which is a sequence
of 2 or more back-to-back interrupts that in most cases
end with the context switch interrupt.

### `interrupt` handlers are not efficient

Although traditional interrupt handlers annotated 
with the `interrupt` attribute may seem a solution for
fast interrupts, they are really fast only if everything
is inlined and no other plain C function is called, otherwise
the entire ABI caller registers must be saved and restored,
including the FP registers, and it must be done repeatedly
for each interrupt, the possibilities for tail chaining and
lazy FP stacking being not realistic.

### Plain C functions are recommended

Plain C interrupt handlers are much better suited for the
common use cases and have the following benefits:

- easier to use in user code
- allow tail chaining without user intervention
- allow lazy FP stacking without user intervention
- save only the ABI caller registers
- save the ABI callee registers only if context switches are triggered

The preferred implementation is with hardware stacking/unstacking,
but cheap devices can also choose to do the stacking/unstacking
in software, together with vectoring, so from a user
point of view they are similar, the interrupt handlers remain
the same plain C functions.
