# Appendix B: Interrupts use cases

Regardless how the interrupts are implemented, the design should be checked
how well accomodates the common use cases.

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

### Cooperative vs preemption

In a cooperative environment, threads deliberately pass control to other
threads by explicitly issuing an `yield()` call, or indirectly by calling
a system function that internally yields.

In a cooperative environment, interrupt handlers are regular handlers, 
they interrupt the current running code (thread or interrupt), 
perform some operations, and return in exactly the same context.

Preemptive environments improve response time by extending some of
the interrut handlers with code that also perform context switches,
such that the interrupt occurs in the context of one thread but 
returns in the context of another thread.

TBD
