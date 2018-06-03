# Appendix B: Interrupts use cases

Regardless how the interrupts are implemented, any architecture design should 
be checked how well the common use cases are accomodated.

## Peripherals vs scheduler interrupts

By design, obsolete microcontroller architectures expected interrupts to be 
triggered only occasionally by peripherals.

Since Cortex-M, interrupts were extensively enhanced with features to
support the implementation of RTOSes, greatly simplifying context switches
and preemption. 

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
