# 12Step-OS-Pico-2H
Porting the sample OS from "12ステップで作る組み込みOS自作入門" (KOZOS) from the H8/3069F to the Raspberry Pi Pico 2 H (RP2350), built on macOS.

# Mac Build Tool
```sh
brew install --cask gcc-arm-embedded   # Official Arm GNU Toolchain (arm-none-eabi-gcc)
brew install picotool                  # ELF-to-UF2 conversion and flashing
brew install minicom lrzsz             # Serial terminal and XMODEM transfer (replacing kermit used in the book)
```
# Pico 2H Memory Map
<img width="540" alt="image" src="https://github.com/user-attachments/assets/0a530eb8-fdcd-4ba6-9a8f-68f4ed54c289" />

On the RP2350 (Arm mode), the ROM, flash, SRAM, and peripherals all share a single 32-bit address space. The Pico 2 H has far more memory than the H8/3069F:

| | H8/3069F (book) | Pico 2 H |
|---|---|---|
| ROM | 512KB internal flash | 4MB external flash (XIP) |
| RAM | 16KB internal | 520KB |

Thanks to **XIP (Execute In Place)**, code runs directly from flash without being copied to RAM. This lets the port keep almost the same structure as the book:

- **Bootloader (kzload):** Placed in flash at `0x1000_0000`, starting with the vector table and the IMAGE_DEF block.
- **OS:** Received over XMODEM as an ELF file, then loaded into SRAM from `0x2000_0000` and executed.
- **SRAM layout:** kzload keeps its own variables and stack near the end of SRAM so they don't overlap the OS load area, the same idea as the RAM split in the book's linker script (`ld.scr`).

# Chapter 1　- "Hello World" Source Cord -

## main.c
```C
#include "defines.h"
#include "board.h"
#include "serial.h"
#include "lib.h"

int main(void)
{
  board_init();  /* Added: required only on Pico 2 H */
  serial_init(SERIAL_DEFAULT_DEVICE);

  puts("Hello World!\n");

  while (1)
    ;

  return 0;
}
```

On the H8/3069F, the CPU and peripherals are clocked from the external crystal right after reset, and the SCI pins are dedicated to serial, so configuring the SCI registers is enough.

On the RP2350, peripherals are stopped after power-on. Before using the UART, `board_init()` does the following:

1. Start the crystal oscillator (XOSC, 12MHz)
2. Feed a clock to the peripherals (`clk_peri`)
3. Release UART, GPIO, and pads from reset
4. Switch GP0/GP1 to UART and clear pad isolation (ISO, new in RP2350)

This is board-wide setup rather than serial-specific, so it lives in a separate file (`board.c`) instead of inside `serial_init()`.

## startup.s
```asm
	.syntax unified
	.cpu cortex-m33
	.thumb

	.section .text
	.global	start
	.type	start, %function
	.thumb_func
start:
	ldr	r0, =0x20082000
	mov	sp, r0
	bl	main
1:	b	1b
```

### `startup.s`: H8 vs Cortex-M33

| Book (H8) | Pico 2 H (Cortex-M33) | Purpose |
|---|---|---|
| `.h8300h` | `.syntax unified`<br>`.cpu cortex-m33`<br>`.thumb` | Target CPU |
| `#.type _start,@function` | `.type start, %function`<br>`.thumb_func` | Declare as a function (**required** on Arm) |
| `mov.l #0xffff00,sp` | `ldr r0, =0x20082000`<br>`mov sp, r0` | Set the stack pointer |
| `jsr @_main` | `bl main` | Call `main` |
| `1: bra 1b` | `1: b 1b` | Infinite loop |

- **Leading `_` in symbol names:** The H8 gcc adds `_` to C function names (`main` → `_main`); the Arm gcc does not.
- **Two instructions for SP:** Arm instructions are fixed-length and can't embed a 32-bit value, so it is loaded into `r0` first.
- **`.thumb_func`:** Without it, the Thumb bit isn't set in the vector table and the CPU faults right after reset.
- **`jsr` vs `bl`:** H8 `jsr` pushes the return address onto the stack; Arm `bl` stores it in the `lr` register.
- **No `.data`/`.bss` initialization yet:** Same as the book's step 1. It will be added in step 3.
