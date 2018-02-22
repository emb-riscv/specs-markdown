# Memory map

Generally the RISC-V microcontroller memory map is implementation specific; the only reserved area 
is a slice of 256 MiB at the end of the memory space.

For 32-bits devices, the reserved system area is **0xF0000000-0xFFFFFFFF**.

For 64-bits devices, the reserved system area is **0xFFFFFFFF'F0000000-0xFFFFFFFF'FFFFFFFF**.

Typical RISC-V microcontroller devices have:

- a **read-only code** area, usually at 0x00000000, but the actual address may be implementation 
specific (typically flash)
- a **read-write data** area (typically RAM)
- an implementation specific **peripheral** area.

Multi-hart devices can share certain memory areas (code or data), but can also have hart-specific 
memory areas, or both shared and specific areas.

## The system control blocks

There are two control blocks, one providing control and status registers common for the entire 
device, and one providing control and status registers for the current hart:

| Base | Top | Name | Description |
|:-----|:----|:-----|-------------|
| 0xF000'0000 | 0xF000'XXXX | `dcb` | The Device Control Block. |
| 0xF0XX'XXXX | 0xF0XX'XXXX | `sysclock` | The Device System Clock. |
| 0xF0XX'XXXX | 0xF0XX'XXXX | `rtclock` | The Device Real-Time Clock. |
| 0xF0XX'XXXX | 0xF0XX'XXXX | `wdog` | The Device Watchdog Timer. |
| 0xF100'0000 | 0xF100'0FFF | `hcb` | The Hart Control Block. |
| 0xF100'2000 | 0xF100'3FFF | `hic` | The Hart Interrupt Controller. |

Each hart has its own separate control block; all HCBs map to the same address, the internal 
logic being able to distinguish between them based on the ID of the hart requesting access;
thus each hart can access only its own hart control block.

(the addresses are preliminary, need more work to find a solution easy to decode)
