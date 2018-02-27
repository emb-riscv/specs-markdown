# Device startup

After reset, all harts in a RISC-V microcontroller start executing code, identified by a 
per-hart **startup block**.

The location of the hart startup block is implementation specific. The typical 
configuration with a single hart has the startup block located at the begining 
of the memory space (usually address 0x00000000).

If multiple harts share a memory area to fetch code (like a flash area), the 
startup blocks are organised as an array located at the begining of the shared 
memory area. If different harts have different memory areas, the startup blocks 
are located at the beginning of each area.

For a RISC-V hart, the minimum information required to start a hart is:

- a pointer to the startup routine
- a pointer to the main stack (`spm`)
- a pointer to the RISC-V global pointer (`gp`)
- a pointer to the exception table

All pointers are xlen bits.

For further extensions, a few words at the end of the startup area are reserved.

> <sup>The pointer to the exception table must be known by the hart before entering 
  the startup code, to catch possible execution faults in the startup code.</sup>

## Usage

With the above definition of a startup block, there is no need for any assembly 
instructions, the entire startup code can be written in C/C++.

```c

extern "C" {

typedef void (*riscv_exception_handler_t)(void);

typedef struct
{
  void (*startup)(void);
  void* main_stack_pointer;
  void* global_pointer;
  riscv_exception_handler_t* exception_handlers;
  void* reserved[4];
} riscv_startup_block_t

riscv_startup_block_t
__attribute__((section(".startup_blocks")))
harts_startup_blocks[] = {
  {
    hart0_startup,
    hart0_stack_pointer,
    hart0_global_pointer,
    hart0_exception_handlers
  },
  {
    hart1_startup,
    hart1_stack_pointer,
    hart1_global_pointer,
    hart1_exception_handlers
  }
};

[[noreturn]] void
hart0_startup(void)
{
  // ...
}

[[noreturn]] void
hart1_startup(void)
{
  // ...
}

} // extern "C"
```

### Prerequisites

The linker script must allocate the `.startup_blocks` section at the implementaton 
specific address (usually 0x00000000).

## Implementation

TODO: define a format to express the pseudocode. Possibly Scala?

After reset, each hart will execute the following code, with 

```
start_hart(int hid) 
{
  // Identify the per-hart startup block.
  addr = (word_size * 8) * hid;
  
  // Clear all hart registers.
  hart[hid].x0 = 0;
  hart[hid].x1 = 0;
  // ...
  // Store the exception pointer in the hart specific register.
  hart[hid].excvta = *(addr + word_size * 3);
  
  // Load global pointer.
  hart[hid].gp = *(addr + word_size * 2);
  // Load main stack pointer.
  hart[hid].sp = *(addr + word_size * 1);
  // Load program counter; this will immediately pass control to the startup code.
  hart[hid].pc = *(addr + word_size * 0);
}
```
