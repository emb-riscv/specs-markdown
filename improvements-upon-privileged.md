# Appendix A: Improvements upon RISC-V privileged

## Rationale

As mentioned in RISC-V Volume I, v2.2, the _"RISC-V is a new instruction set architecture (ISA) that was originally designed to support computer architecture research and education. ... The RISC-V manual is structured in two volumes. This volume covers the user-level ISA design, including optional ISA extensions. The second volume provides the privileged architecture."_

The RISC-V Volume II, v1.10, mentions: _"... This document describes the RISC-V privileged architecture, which covers all aspects of RISC-V systems beyond the user-level ISA, including privileged instructions as well as additional functionality required for running operating systems and attaching external devices."_

This is great news for the GNU/Linux community and for the academia, but attempts to identify in the RISC-V specs how the new design meets the requirements of bare-metal embedded devices, were not very successful; browsing the two docs revealed only some references to Tensilica and ARC (probably not the most successful architectures), and some incomplete specs for **the RV32E subset** (which halves the number of general registers, do not support hardware floating point and makes some counter instructions optional).

According to the privileged specs, **RISC-V embedded systems share the exact same definitions as systems running Unix-like operating systems, but they do not include the "S" (Supervisor) mode features**.

This strategy does not work very well for real-time systems; for example, **in the RISC-V interrupt model**, without special measures, **interrupts remain disabled while executing interrupt handlers**. This may be acceptable for Linux kernels, but for hard real-time systems this is generally a no-go, since **interrupt latency** may end up well above tolerable limits.

### Relax the requirement for the privileged specs

The RISC-V Volume II, v1.10, mentions: _"... the entire privileged-level design described in this document could be replaced with an entirely different privileged-level design without changing the user-level ISA, and possibly without even changing the ABI. In particular, this privileged specification was designed to run existing popular operating systems, and so embodies the conventional level-based protection model. Alternate privileged specifications could embody other more flexible protection-domain models."_

So, at least in theory, there should be possible to extend the specs, but in practice it is not clear how exactly this can be done. Ideally, the Volume I should not explicitly refer to Volume II, or should refer to it as optional, leaving room for a complementary specification better suited for devices that do not need to run Unix-like operating systems.

This is kind of silly, since the RISC-V ISA specs provide a very high degree of flexibility allowing for custom extensions for the instruction set, but they are still very rigid by insisting that all these devices should be able to run Unix-like operating systems.

### The virtual memory dividing line

Although some projects try to challenge this, **Unix-like operating systems do need virtual memory to operate properly**.

After long considerations, the conclusion was that the common and logical dividing line between the RISC-V privileged profile and a possible RISC-V microcontroller profile is the use of virtual memory, not needeed for microcontrollers.

## Criticism

### Fragmentation would break upward compatibility

While discussing the opportunity for a new RISC-V embedded profile, the most common concern raised was that migrating an applications written for a microcontroller to a larger application class core would be more difficult.

Well, yes, in theory it might be possible to design a board in such a way to allow to swap in a bigger core, and to design the application in such a way to ignore the MMU and the supervisor mode and continue to use a RTOS; the application will probably run faster due to the improved pipelines and core clock rates, but there are several practical issues:

- in industrial embedded applications the processor selection is not based on the architecture (which in the majority of cases is Cortex-M only), but on the available on-chip peripherals; it is very unlikely to find an application class core with the desired peripherals available on a microcontroller;
- application class cores generally do not have internal flash/ram, requiring external chips; external memory chips require lots of address and data pins, which mean large BGA chips, larger & more complex PCBs, and generally higher costs. 

So this concern is not realistic, and not accepting a distinct microcontroller profile, optimised for real-time applications simply for maintaining compatibility with the privileged specs is not a beneficial approach.

## Examples

... TBD
