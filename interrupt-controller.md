# The Hart Interrupt Controller (HIC)

The RISC-V microcontroller profile provides a nested vectored interrupt controller as part of 
the common specifications.

Each hart may be able to process its own set of interrupts, independent from the other harts. 
Only hart 0 is required to implement a HIC; additional interrupt controllers in all other 
harts are optional and implementation specific.

> <sup>Hart real-time devices may dedicate separate harts to process fast interrupts. At the limit,
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

| Name | Offset | Width | Type | Description |
|:-----|:-------|:------|:-----|-------------|
| irqtab | 0x0000 | xlen | rw | Address of the interrupts table. |
| prio | 0x0010 | 32b | rw | Interrupt priority threshold. |
| interrupts[] | 0x1000 | 32b * N | rw | Array of interrupt control registers. |

Total size: 0x2000.

## Interrupts table address (irqtab)

The address of the interrupts dispatch table. The table is an array of addresses (xlen size elements) pointing to interrupt handlers (C/C++ functions).

If not set and an interrupt occurs, an exception is triggered (TODO: what exception?).

## Interrupts priority threshold register (prio)

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [7-0] | `prio` | rw | The interrupt priority threshold. |
| [31-8] ||| Reserved. |

## Interrupt control register

Each interrupt has a small per-hart set of status and configuration attributes:

* enabled: interrupts can either be disabled (default) or enabled 
* pending: interrupts can either be pending (a request is waiting to be served) or not
pending
* active: interrupts can either be in an active (being served) or inactive state
* prio: interrupt priority

To store and control these attributes, each interrupt has a per-hart 32-bits status and 
control register with the following fields:


| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [7-0] | `prio` | rw | If non zero, the interrupt priority. |
| [15-8] | `status`| r | Status bits |
| [23-16] | `set` | 1s | Set bits. |
| [31-24] | `clear` | 1c | Clear bits. |

The `status` bits:

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [0] | `enabled` | r | Enabled status bit; 1 if the interupt is enabled. |
| [1] | `pending` | r | Pending status bit; 1 if the interupt is pending. |
| [2] | `active` | r | Active status bit; 1 if the interupt is active. | 
| [7-3] ||| Reserved |

The `set` bits:

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [0] | `enabled` | 1s | When 1 is written, the `enabled` status bit is set. |
| [1] | `pending` | 1s | When 1 is written, the `pending` status bit is set. |
| [7-2] ||| Reserved |

The `clear` bits:

| Bits | Name | Type | Description |
|:-----|:-----|:-----|-------------|
| [0] | `enabled` | 1s | When 1 is written, the `enabled` status bit is cleared. |
| [1] | `pending` | 1s | When 1 is written, the `pending` status bit is cleared. |
| [7-2] ||| Reserved |

> <sup>The alternative to packing all status and control bits related to an interrupt 
  in a word would be to have separate multi-word fields with status, enable, disable,
  set pending, clear pending, active bits. It was considered that the packed solution
  is easier to use in software.</sup>
  
### Usage

Individual interrupts are enabled by setting the `status.enabled` bit and are disabled by clearing the `enabled` bit. To be effective, interrupts must also have non-zero priorities.

```c
hic.interrupts[7].prio = 0xC0; // A byte write cycle.
hic.interrupts[7].set = INTERRUPTS_SET_ENABLED; // A byte write cycle.

hcb.interrupts[7].clear = INTERRUPTS_CLEAR_ENABLED; // A byte write cycle.
```

Interrupts can be programatically set to be pending by writing 1 in the `status.pending` field; the pending status can be cleared by writing 1 to the `clear_pend` field.

```c
hcb.interrupts[7].set = INTERRUPTS_SET_PENDING; // A byte write cycle.
hcb.interrupts[7].clear = INTERRUPTS_CLEAR_PENDING; // A byte write cycle.
```

To check the status bits:

```c
if (hcb.interrupts[7].status & INTERRUPTS_STATUS_PENDING) { // A byte read cycle.
  // ...
}
```
