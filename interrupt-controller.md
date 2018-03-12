# The Hart Interrupt Controller (HIC)

The RISC-V microcontroller profile provides a nested vectored interrupt controller as part of 
the common specifications.

Each hart may be able to process its own set of interrupts, independent from the other harts. 
Only hart 0 is required to implement a HIC; additional interrupt controllers in all other 
harts are optional and implementation specific.

> <sup>Hard real-time devices may dedicate separate harts to process fast interrupts. 
  It is possible to wire all interrupts to all harts, and decide in software which interrupts
  are processed by each hart.</sup>

## Features

The HIC supports the following features:

- HIC interrupts can be enabled and disabled by writing to their corresponding `status.enabled` or 
`status.clearenabled` bit fields, using a write-1-to-enable and write-1-to-clear policy.

  When an interrupt is disabled, interrupt assertion causes the interrupt to become pending, but the interrupt 
cannot become active. If an interrupt is active when it is disabled, it remains in the active state until 
this is cleared by a reset or an exception return. Clearing the enable bit prevents any new activation of 
the associated interrupt.

  An implementation can hard-wire interrupt enable bits to zero if the associated interrupt line does not 
exist, or hard-wired them to one if the associated interrupt line cannot be disabled.

- the pending state of HIC interrupts can set or removed by software using 
the `status.pending` and `status.clearpending` bit fields. The registers use a write-1-to-enable and 
write-1-to-clear policy. Writing 1 to a bit in the `status.clearpending` bit field has no effect on the 
execution status of an active interrupt.

  It is implementation specific for each interrupt line supported, whether an interrupt supports either or both 
setting and clearing of the associated pending state under software control.

- status bits are provided to allow software to determine whether an interrupt is active, pending, or enabled.
- HIC interrupts are prioritized by updating a priority field. Priorities are maintained according to the RISC-V 
prioritization scheme.
- HIC supports a maximum of 1024 interrupts.

## Memory map

| Offset | Name | Width | Type | Reset | Description |
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `interrupts[]` | 32b * 2 * N | rw | 0x00000000 | Array of interrupt control registers. |

The number of interrupts (N) is implementation specific, but no higher than 1024, including the system interrupts.

Total size: 0x2000.

## Per interrupt registers

Each interrupt has a small per-hart set of status and configuration attributes:

* `enabled`: interrupts can either be disabled (default) or enabled 
* `pending`: interrupts can either be pending (a request is waiting to be served) or not
pending
* `active`: interrupts can either be in an active (being served) or inactive state
* `prio`: interrupt priority

To store and control these attributes, each interrupt has two 32-bits registers:

| Offset | Name | Width | Type | Reset | Description |
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `prio` | 32b | rw | 0x00000000 | The interrupt priority register. |
| 0x0004 | `status` | 32b | rw | 0x00000000 | The interrupt status and control register. |

The `prio` register has the the following content:

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [N:0] | `prio` | rw | 0 | The interrupt priority. |
| [(xlen-1):(N+1)] | | | | Reserved. |

N is the number of bits required to store the maximum priority level, and is implementation 
specific. It must match the number of bits used by the `iprioth` CSR.

The `status` register has the following content: 

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | `enabled` | rw1s | 0 | Enabled status bit; 1 if the interrupt is enabled.<br>When 1 is written, the `enabled` bit is set. |
| [1] | `pending` | rw1s | 0 | Pending status bit; 1 if the interrupt is pending.<br>When 1 is written, the `pending` bit is set. |
| [2] | `active` | r | 0 | Active status bit; 1 if the interrupt is active. | 
| [3] |||| Reserved |
| [4] | `clearenabled` | w1c | | When 1 is written, the `enabled` status bit is cleared. |
| [5] | `clearpending` | w1c | | When 1 is written, the `pending` status bit is cleared. |
| [31:6] |||| Reserved |

> <sup>The alternative to packing all status and control bits related to an interrupt 
  in two words would be to have separate multi-word fields with status, enable, disable,
  set pending, clear pending, active bits. It was considered that the packed solution
  is easier to use in software.</sup>
  
> <sup>[JB] Why use separate bits for enabling and disabling interrupts? Why not 
use the same write-1-to-enable and write-0-to-clear? ... it is not the way 
assignment works anywhere else in C. [ilg] C programmers are very much used 
  to different semantics when accessing peripheral registers, actually most 
  real peripherals in modern devices use write-1-to-set/clear bits, so this 
  not surprise anybody. As for keeping the language semantics, in C++ all 
  operators can be redefined, so they can implement any semantics is 
  required. </sup>

## Usage

Individual interrupts are enabled by setting the `status.enabled` bit and are disabled by writing 1 in the `status.clearenabled` bit. To be effective, interrupts must also have non-zero priorities.

```c
hic.interrupts[7].prio = 7;
hic.interrupts[7].status = INTERRUPTS_SET_ENABLED;

hcb.interrupts[7].status = INTERRUPTS_CLEAR_ENABLED;
```

Interrupts can be programmatically set to be pending by writing 1 in the `status.pending` field; the pending status can be cleared by writing 1 to the `status.clearpending` bit.

```c
hcb.interrupts[7].status = INTERRUPTS_SET_PENDING;
hcb.interrupts[7].status = INTERRUPTS_CLEAR_PENDING;
```

To check the status bits:

```c
if (hcb.interrupts[7].status & INTERRUPTS_STATUS_PENDING) {
  // ...
}
```

## Alternate proposal

[JB] Each hart has a fixed set of interrupt vectors. For each interrupt 
source, a register exists that defines which vector receives interrupts 
from that source. Effectively, the hart has some number of IRQ lines 
and interrupt sources are assigned manually to IRQ lines. The IRQ lines 
have a fixed priority, based on the interrupt number. 

If multiple 
peripherals are assigned to the same vector, then the ISR for that 
vector must poll each of the peripherals assigned to that vector to 
determine the cause of the interrupt. 

This also limits interrupt nesting, since only a higher-priority 
interrupt (or an exception) can interrupt an ISR, there can be at most 
2\*PRIORITY_LEVELS nested interrupts, if every ISR is interrupted in an 
exception handler and exception handlers do not themselves raise 
exceptions. 

> <sup>[ilg] The only advantage to be noted is that it limits nesting. The disadvantages are: increased software complexity, increased latency, more complicated to maintain (changing the priority in the first case requires only a write to the priority register, while in the second case it is also necessary to move the test and the call from one intermediate handler to the other), possible out-of-sync cases, when the test is not in the right handler.</sup>
