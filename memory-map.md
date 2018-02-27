# Memory map

Generally the RISC-V microcontroller memory map is implementation specific; the only reserved area 
in the RISC-V microcontroller profile is a slice at the end of the memory space, called the 
**system control area**. 

Typical RISC-V microcontroller devices have:

- a **read-only code** area, usually at 0x00000000, but the actual address may be implementation 
specific (typically flash)
- a **read-write data** area (typically RAM)
- an implementation specific **peripheral** area.

Multi-hart devices can share certain memory areas (code or data), but can also have hart-specific 
memory areas, or both shared and specific areas.

## The system control area

The system control area is a slice of 256 MiB at the end of the memory space. This area 
must have the execute permissions removed, and attempts to execute code from it must trigger 
exceptions (Which one?)

For 32-bits devices, the system control area is **0xF0000000-0xFFFFFFFF**.

For 64-bits devices, the system control area is **0xFFFFFFFF'F0000000-0xFFFFFFFF'FFFFFFFF**.

The system control area is implemented as a set of memory-mapped address spaces, some providing control and status registers common for the entire 
device, and some providing control and status registers for the current hart:

| Base | Top | Name | Description |
|:-----|:----|:-----|-------------|
| 0xF000'0000 | 0xF000'0FFF | `dcb` | The Device Control Block. |
| 0xF000'1000 | 0xF000'1FFF | `sysclock` | The Device System Clock. |
| 0xF000'2000 | 0xF000'2FFF | `rtclock` | The Device Real-Time Clock. |
| 0xF000'3000 | 0xF000'3FFF | `wdog` | The Device Watchdog Timer. |
| | | | |
| 0xF100'0000 | 0xF100'0FFF | `hcb` | The Hart Control Block. |
| 0xF100'2000 | 0xF100'3FFF | `hic` | The Hart Interrupt Controller. |

Each hart has its own separate control block; all HCBs map to the same address, the internal 
logic being able to distinguish between them based on the ID of the hart requesting access;
thus each hart can access only its own hart control block. 

Same for the Hart Interrupt Controller.

(the addresses are preliminary, need more work to find a solution easy to decode)
