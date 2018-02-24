# The Hart Interrupt Controller (HIC)

The RISC-V microcontroller profile provides a nested vectored interrupt controller as part of 
the common specifications.

Each hart may be able to process its own set of interrupts, independent from the other harts. 
Only hart 0 is required to implement a HIC; additional interrupt controllers in all other 
harts are optional and implementation specific.

> <sup>Hard real-time devices may dedicate separate harts to process fast interrupts. At the limit,
  it is possible to wire all interrupts to all harts, and decide in software which interrupts
  are processed by each hart.</sup>

## Features

The HIC supports the following features:

- HIC interrupts can be enabled and disabled by writing to their corresponding `set.enabled` or 
`clear.enabled` bit fields, using a write-1-to-enable and write-1-to-clear policy.

  When an interrupt is disabled, interrupt assertion causes the interrupt to become pending, but the interrupt 
cannot become active. If an interrupt is active when it is disabled, it remains in the active state until 
this is cleared by a reset or an exception return. Clearing the enable bit prevents any new activation of 
the associated interrupt.

  An implementation can hard-wire interrupt enable bits to zero if the associated interrupt line does not 
exist, or hard-wired them to one if the associated interrupt line cannot be disabled.

- the pending state of HIC interrupts can set or removed by software using 
the `set.pending` and `clear.pending` bit fields. The registers use a write-1-to-enable and 
write-1-to-clear policy. Writing 1 to a bit in the `clear.pending` bit field has no effect on the 
execution status of an active interrupt.

  It is implementation specific for each interrupt line supported, whether an interrupt supports either or both 
setting and clearing of the associated pending state under software control.

- status bits are provided to allow software to determine whether an interrupt is active, pending, or enabled.
- HIC interrupts are prioritized by updating an 8-bit field. Priorities are maintained according to the RISC-V 
prioritization scheme
- HIC supports a maximum 1024 interrupts.

## Memory map

| Offset | Name | Width | Type | Reset | Description |
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `intvta` | xlen | rw | 0x00000000 | Address of the interrupts vector table. |
| 0x1000 | `interrupts[]` | 32b * N | rw | 0x00000000 | Array of interrupt control registers. |

The number of interrupts (N) is implementation specific, but no higher than 1024, including the system interrupts.

Total size: 0x2000.

## Interrupts vector table address (intvta)

The address of the interrupts dispatch table. The table is an array of addresses (xlen size elements) pointing to interrupt handlers (C/C++ functions).

If not set (i.e. 0x0) and an interrupt occurs, an exception is triggered (TODO: what exception?).

## Interrupt control register

Each interrupt has a small per-hart set of status and configuration attributes:

* `enabled`: interrupts can either be disabled (default) or enabled 
* `pending`: interrupts can either be pending (a request is waiting to be served) or not
pending
* `active`: interrupts can either be in an active (being served) or inactive state
* `prio`: interrupt priority

To store and control these attributes, each interrupt has a per-hart two 32-bits registers:

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
specific.

The `status` register has the following content: 

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | `enabled` | rw1s | 0 | Enabled status bit; 1 if the interupt is enabled.<br>When 1 is written, the `enabled` bit is set. |
| [1] | `pending` | rw1s | 0 | Pending status bit; 1 if the interupt is pending.<br>When 1 is written, the `pending` bit is set. |
| [2] | `active` | r | 0 | Active status bit; 1 if the interupt is active. | 
| [3] |||| Reserved |
| [4] | `clearenabled` | w1c | | When 1 is written, the `enabled` status bit is cleared. |
| [5] | `clearpending` | w1c | | When 1 is written, the `pending` status bit is cleared. |
| [31:6] |||| Reserved |

> <sup>The alternative to packing all status and control bits related to an interrupt 
  in two words would be to have separate multi-word fields with status, enable, disable,
  set pending, clear pending, active bits. It was considered that the packed solution
  is easier to use in software.</sup>
  
### Usage

Individual interrupts are enabled by setting the `status.enabled` bit and are disabled by clearing the `status.clearenabled` bit. To be effective, interrupts must also have non-zero priorities.

```c
hic.interrupts[7].prio = 7;
hic.interrupts[7].status = INTERRUPTS_SET_ENABLED;

hcb.interrupts[7].status = INTERRUPTS_CLEAR_ENABLED;
```

Interrupts can be programatically set to be pending by writing 1 in the `status.pending` field; the pending status can be cleared by writing 1 to the `status.clearpending` bit.

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
