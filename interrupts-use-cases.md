# Appendix B: Interrupts use cases

Regardless how the interrupts are implemented, any architecture design should 
be checked how well the common use cases are accomodated.

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

The problem with this approach is that on a RISC-V device 
with the current ABI, the number of
registers required to be saved by the caller is large, 
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


### Context switches

In a multithreaded environment, a context switches is generally 
a sequence of operations performing the following steps:

- interrupt the current thread
- save the state of the current thread in the thread control block (TCB)
- select the next thread to run
- restore the state of the new thread from the selected TCB
- resume execution in the context of the new thread

### Cooperative vs preemptive

In a cooperative environment, threads deliberately pass control to other
threads either by directly issuing an `yield()` call, or indirectly 
by calling a system function that internally yields.

In a cooperative environment, user interrupt handlers are regular handlers, 
they interrupt the current running code (thread or interrupt), 
perform some operations, and return in exactly the same context.

Preemptive environments improve response time by extending some of
the interrut handlers with code that also performs context switches,
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

### Dedicated context switch interrupt

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

### Triggering a context switch

With such a dedicated interrupt, triggering a context switch is as simple 
as pending a software interrupt:

```c
hcb.interrupts[CONTEXT_SWITCH_INTERRUPT_NUMBER].status = INTERRUPTS_SET_PENDING;
```

Pending a context switch interrupt can be performed either in other interrupt
handlers (and in this case the switch occurs after all interrupts are 
completed), or in thread mode, in the `yield()` function, and in this case
the switch is performed as soon as interrupts are enabled.





TBD
