# Device startup

After reset, all harts in a RISC-V microcontroller start executing code, identified by a per-hart startup block.

The startup blocks are organised as an array located at the begining of the memory space (address 0x00000000).

For a RISC-V hart, the minimum content of a startup block is:

- a pointer to the startup routine
- a pointer to the main stack (SP)
- a pointer to the RISC-V global pointer (GP)
- a pointer to the exception table

The pointer to the execption table must be known by the hart before entering the startup code, to catch possible execution faults in the startup code.

## Usage

With the above definition of a startup block, there is no need for any assembly instructions, the entire startup code can be written in C/C++.

```cpp
#if defined __cplusplus
extern "C"
{
#endif

typedef void (*riscv_exception_handler_t)(void);

typedef struct
{
  void (*startup)(void);
  void* main_stack_pointer;
  void* global_pointer;
  riscv_exception_handler_t* exception_handlers;
  void* reserved[4];
} riscv_arch_startup_block_t

riscv_arch_startup_block_t
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

#if defined __cplusplus
}
#endif
```

### Implementation

TODO: define a format to express the pseudocode. Possibly Scala?

```
start_hart(int i) 
{
  addr = (word_size * 8) * i;
  
  // Clear all registers
  x0 = 0;
  x1 = 0;
  // ...
  // Store the exception pointer in the hart specific register
  sys.hart[i].excptr = *(addr + word_size * 3);
  
  // Load global pointer.
  gp = *(addr + word_size * 2);
  // Load main stack pointer.
  sp = *(addr + word_size * 1);
  // Load program counter; this will immediately pass control to the startup.
  pc = *(addr + word_size * 0);
}
```
