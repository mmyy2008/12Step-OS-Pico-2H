# 12Step-OS-Pico-2H
Porting the sample OS from "12ステップで作る組み込みOS自作入門" (KOZOS) from the H8/3069F to the Raspberry Pi Pico 2 H (RP2350), built on macOS.

# Build Tools (macOS)
```sh
brew install --cask gcc-arm-embedded   # Official Arm GNU Toolchain (arm-none-eabi-gcc)
brew install picotool                  # ELF-to-UF2 conversion and flashing
brew install minicom lrzsz             # Serial terminal and XMODEM transfer (replacing kermit used in the book)
```
# Pico 2 H Memory Map
<img width="540" alt="Pico 2 H memory map" src="https://github.com/user-attachments/assets/0a530eb8-fdcd-4ba6-9a8f-68f4ed54c289" />

On the RP2350 (Arm mode), the ROM, flash, SRAM, and peripherals all share a single 32-bit address space. The Pico 2 H has far more memory than the H8/3069F:

| | H8/3069F (book) | Pico 2 H |
|---|---|---|
| ROM | 512KB internal flash | 4MB external flash (XIP) |
| RAM | 16KB internal | 520KB |

Thanks to **XIP (Execute In Place)**, code runs directly from flash without being copied to RAM. This lets the port keep almost the same structure as the book:

- **Bootloader (kzload):** Placed in flash at `0x1000_0000`, starting with the vector table and the IMAGE_DEF block.
- **OS:** Received over XMODEM as an ELF file, then loaded into SRAM from `0x2000_0000` and executed.
- **SRAM layout (from step 4 onward):** kzload keeps its own variables and stack near the end of SRAM so they don't overlap the OS load area, the same idea as the RAM split in the book's linker script (`ld.scr`).

# Step 1: Hello World

## main.c
```c
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

## board.c

```c
/*
 * Board init for Pico 2 H (not in the book: the H8 needs none of this)
 *
 *   1. Start the 12MHz crystal (XOSC)
 *   2. clk_ref <- XOSC, clk_sys <- clk_ref, clk_peri <- clk_sys  (all 12MHz)
 *   3. Take IO_BANK0 / PADS_BANK0 / UART0 out of reset
 *   4. Route GP0 = UART0 TX, GP1 = UART0 RX
 *
 * No PLL is used, to keep step 1 as simple as possible.
 */
#include "defines.h"
#include "rp2350.h"
#include "board.h"

static void xosc_init(void)
{
  /* Startup delay in units of 256 cycles (generous: about 20ms) */
  REG32(XOSC_STARTUP) = 1000;
  REG32(XOSC_CTRL) = XOSC_CTRL_FREQ_RANGE_1_15MHZ | XOSC_CTRL_ENABLE;
  while (!(REG32(XOSC_STATUS) & XOSC_STATUS_STABLE))
    ;
}

static void clocks_init(void)
{
  /* clk_ref <- XOSC (glitchless mux; SELECTED is one-hot) */
  REG32(CLK_REF_CTRL) = CLK_REF_SRC_XOSC;
  while (REG32(CLK_REF_SELECTED) != (1UL << CLK_REF_SRC_XOSC))
    ;

  /* clk_sys <- clk_ref */
  REG32(CLK_SYS_CTRL) = CLK_SYS_SRC_CLK_REF;
  while (REG32(CLK_SYS_SELECTED) != (1UL << CLK_SYS_SRC_CLK_REF))
    ;

  /* clk_peri (feeds the UART) <- clk_sys, then enable */
  REG32(CLK_PERI_CTRL) = 0;
  REG32(CLK_PERI_CTRL) = CLK_PERI_AUXSRC_SYS | CLK_PERI_ENABLE;
}

static void unreset(uint32 bits)
{
  REG32(RESETS_RESET + REG_ALIAS_CLR) = bits;
  while ((REG32(RESETS_RESET_DONE) & bits) != bits)
    ;
}

static void gpio_set_uart(int pin, uint32 pad_extra)
{
  /* Input enable on, output disable off */
  REG32(PADS_GPIO(pin)) = (REG32(PADS_GPIO(pin)) & ~PADS_OD) | PADS_IE | pad_extra;
  REG32(GPIO_CTRL(pin)) = GPIO_FUNC_UART;
  /* Release pad isolation (forget this and nothing comes out of the pin) */
  REG32(PADS_GPIO(pin) + REG_ALIAS_CLR) = PADS_ISO;
}

void board_init(void)
{
  xosc_init();
  clocks_init();
  unreset(RESETS_IO_BANK0 | RESETS_PADS_BANK0 | RESETS_UART0);
  gpio_set_uart(0, 0);         /* GP0: UART0 TX */
  gpio_set_uart(1, PADS_PUE);  /* GP1: UART0 RX (pull-up) */
}
```

| Function | What it does |
|---|---|
| `xosc_init()` | Starts the 12MHz crystal oscillator and waits until it is stable |
| `clocks_init()` | Switches `clk_ref` and `clk_sys` to the crystal, then enables `clk_peri` for the UART |
| `unreset()` | Releases peripherals from reset and waits until they are ready |
| `gpio_set_uart()` | Enables the pad input, selects the UART function, and clears pad isolation |

- **No PLL:** Everything runs at 12MHz to keep step 1 simple.
- **Atomic register aliases:** Writing to `base + 0x3000` clears only the given bits (used for RESETS and pad ISO), so no read-modify-write is needed.
- **Pad isolation (ISO):** New in RP2350. Pads are isolated after reset, so a pin outputs nothing until ISO is cleared.
- Register definitions are in [`rp2350.h`](01/bootload/rp2350.h). `board.h` only declares `board_init()`.

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

- **Leading `_` in symbol names:** The H8 gcc adds `_` to C function names (`main` → `_main`); the Arm gcc does not. That's why the book's `vector.c` refers to `start` while its `startup.s` defines `_start`.
- **Two instructions for SP:** Arm instructions are fixed-length and can't embed a 32-bit value, so it is loaded into `r0` first.
- **`.thumb_func`:** Without it, the Thumb bit isn't set in the vector table and the CPU faults right after reset.
- **`jsr` vs `bl`:** H8 `jsr` pushes the return address onto the stack; Arm `bl` stores it in the `lr` register.
- **No `.data`/`.bss` initialization yet:** Same as the book's step 1. It will be added in step 3.

## vector.c

```c
/*
 * Vector table + IMAGE_DEF block
 *
 * H8: vectors[] is an array of handler addresses starting at address 0.
 * Cortex-M: word 0 is the initial stack pointer, word 1 the reset handler.
 * The RP2350 boot ROM loads SP and PC from here.
 */
#include "defines.h"

extern void start(void);

void (*vectors[])(void) __attribute__((section(".vectors"), used)) = {
  (void (*)(void))0x20082000, /* 0: initial SP (end of SRAM) */
  start,                   /* 1: Reset */
};

/*
 * IMAGE_DEF block: the RP2350 boot ROM only boots an image that has
 * this marker within the first 4KB of flash.
 * "Secure Arm executable for RP2350" (minimum IMAGE_DEF in the datasheet).
 */
const uint32 image_def[] __attribute__((section(".embedded_block"), used)) = {
  0xffffded3, /* PICOBIN_BLOCK_MARKER_START */
  0x10210142, /* IMAGE_TYPE: EXE | Secure | Arm | RP2350 */
  0x000001ff, /* LAST item, size 1 */
  0x00000000, /* Link to next block: 0 = points to itself */
  0xab123579, /* PICOBIN_BLOCK_MARKER_END */
};
```

### `vector.c`: H8 vs Cortex-M33

| | Book (H8) | Pico 2 H (Cortex-M33) |
|---|---|---|
| Entry 0 | Reset handler (`start`) | Initial stack pointer |
| Entry 1 | Interrupt vector | Reset handler (`start`) |
| Number of entries | 64, unused ones filled with `NULL` | 2 (only what step 1 needs) |
| Placement | `vector.o(.data)` in `ld.scr` | `section(".vectors")` attribute + `vector.o(.vectors)` in `ld.scr` |
| Extra | None | `image_def[]`: IMAGE_DEF block required by the RP2350 boot ROM |

- On Cortex-M, the CPU loads both the initial SP and the reset handler from the vector table at reset.
- The full exception and interrupt table (NMI, HardFault, UART IRQ, etc.) will be added in step 7.

## lib.h & lib.c

`lib.h` and `lib.c` only call the serial driver function (`serial_send_byte()`) and never touch the hardware directly, so they don't need to change when the CPU changes.

## serial.h

`serial.h` is the interface to the hardware-dependent `serial.c`. Only the implementation in `serial.c` was rewritten; the function names and arguments (just a device index) contain nothing H8-specific, so the interface stays the same.

## serial.c

```c
/*
 * Serial driver: H8 SCI -> RP2350 UART (ARM PL011)
 * Same structure and function names as the book.
 */
#include "defines.h"
#include "rp2350.h"
#include "serial.h"

#define SERIAL_DEVICE_NUM 2

#define UART_CLK_HZ 12000000UL  /* clk_peri (see board.c) */
#define UART_BAUD   115200UL    /* The book uses 9600; change here if you like */

struct rp2350_uart {
  volatile uint32 dr;     /* 0x00 data */
  volatile uint32 rsr;    /* 0x04 receive status / error clear */
  volatile uint32 _res0[4];
  volatile uint32 fr;     /* 0x18 flags */
  volatile uint32 _res1;
  volatile uint32 ilpr;   /* 0x20 */
  volatile uint32 ibrd;   /* 0x24 integer baud divisor */
  volatile uint32 fbrd;   /* 0x28 fractional baud divisor */
  volatile uint32 lcr_h;  /* 0x2c line control */
  volatile uint32 cr;     /* 0x30 control */
  volatile uint32 ifls;   /* 0x34 */
  volatile uint32 imsc;   /* 0x38 interrupt mask */
};

#define UART_FR_TXFF  (1UL << 5)  /* TX FIFO full */
#define UART_FR_RXFE  (1UL << 4)  /* RX FIFO empty (used in step 2) */
#define UART_LCR_H_WLEN_8 (3UL << 5)
#define UART_LCR_H_FEN    (1UL << 4)
#define UART_CR_UARTEN (1UL << 0)
#define UART_CR_TXE    (1UL << 8)
#define UART_CR_RXE    (1UL << 9)

static struct {
  volatile struct rp2350_uart *uart;
} regs[SERIAL_DEVICE_NUM] = {
  { (volatile struct rp2350_uart *)UART0_BASE },
  { (volatile struct rp2350_uart *)UART1_BASE },
};

/* Initialize device */
int serial_init(int index)
{
  volatile struct rp2350_uart *uart = regs[index].uart;

  /* Baud divisor = UART_CLK / (16 * baud), fraction in 1/64 units */
  uint32 div = (8 * UART_CLK_HZ) / UART_BAUD;  /* = divisor * 128 */
  uint32 ibrd = div >> 7;
  uint32 fbrd = ((div & 0x7f) + 1) / 2;

  uart->cr = 0;
  uart->ibrd = ibrd;
  uart->fbrd = fbrd;
  uart->lcr_h = UART_LCR_H_WLEN_8 | UART_LCR_H_FEN; /* 8N1, FIFO on (also latches the divisor) */
  uart->imsc = 0;                                   /* No interrupts */
  uart->cr = UART_CR_UARTEN | UART_CR_TXE | UART_CR_RXE;

  return 0;
}

/* Ready to send? */
int serial_is_send_enable(int index)
{
  volatile struct rp2350_uart *uart = regs[index].uart;
  return !(uart->fr & UART_FR_TXFF);
}

/* Send one byte */
int serial_send_byte(int index, unsigned char c)
{
  volatile struct rp2350_uart *uart = regs[index].uart;

  /* Wait until the TX FIFO has room */
  while (!serial_is_send_enable(index))
    ;
  uart->dr = c;

  return 0;
}
```

The functions seen from `lib.c` are the same as the book, but the hardware underneath changes from the H8 SCI to the RP2350 UART (Arm PL011).

### `serial.c`: Book vs Pico 2 H

| | Book (H8 SCI) | Pico 2 H (RP2350 UART) |
|---|---|---|
| Register struct | `struct h8_3069f_sci` (SMR, BRR, SCR, TDR, SSR, ...) | `struct rp2350_uart` (DR, FR, IBRD, FBRD, LCR_H, CR, ...) |
| Devices | 3 (SCI0–2) | 2 (UART0, 1) |
| Baud rate | One value in BRR (64 for 9600 bps @ 20MHz) | Integer IBRD + fractional FBRD (6 and 33 for 115200 bps @ 12MHz) |
| Ready to send | TDRE bit in SSR is 1 | TXFF bit (TX FIFO full) in FR is 0 |
| Send | Write TDR, then clear TDRE | Just write DR (no flag handling) |
| FIFO | None | 32 bytes |

## defines.h

```c
#ifndef _DEFINES_H_INCLUDED_
#define _DEFINES_H_INCLUDED_

#define NULL ((void *)0)
#define SERIAL_DEFAULT_DEVICE 0

typedef unsigned char  uint8;
typedef unsigned short uint16;
typedef unsigned long  uint32;

#endif
```

### `defines.h`: Book vs Pico 2 H

| | Book (H8) | Pico 2 H |
|---|---|---|
| `SERIAL_DEFAULT_DEVICE` | `1` (SCI1, wired to the board's serial connector) | `0` (UART0 on GP0/GP1) |

The type definitions are unchanged: `unsigned long` is 32 bits on both the H8 and Arm gcc toolchains, so `uint32` still works as-is.

## ld.scr

```ld
OUTPUT_FORMAT("elf32-littlearm")
OUTPUT_ARCH(arm)
ENTRY(start)

SECTIONS
{
	. = 0x10000000;

	.vectors : {
		vector.o(.vectors)
	}

	.text : {
		vector.o(.embedded_block)
		*(.text)
	}

	.rodata : {
		*(.strings)
		*(.rodata)
		*(.rodata.*)
	}

	.data : {
		*(.data)
	}

	.bss : {
		*(.bss)
		*(COMMON)
	}
}
```

### `ld.scr`: Book vs Pico 2 H

| | Book (H8) | Pico 2 H |
|---|---|---|
| Output format | `h8300h` (`OUTPUT_FORMAT` commented out) | `elf32-littlearm`, `arm` |
| Entry | `"_start"` | `start` |
| Start address | `0x0` | `0x10000000` (top of flash) |
| Vectors | `vector.o(.data)` | `vector.o(.vectors)` |
| Added | — | `vector.o(.embedded_block)`: places the IMAGE_DEF block right after the vectors |

- Like the book, `.data` and `.bss` are placed in flash, so global variables can't be written yet. This is fixed in step 3.

## Makefile

```makefile
PREFIX  = $(shell brew --prefix)
ARCH    = arm-none-eabi
BINDIR  = $(PREFIX)/bin
ADDNAME = $(ARCH)-

AR      = $(BINDIR)/$(ADDNAME)ar
AS      = $(BINDIR)/$(ADDNAME)as
CC      = $(BINDIR)/$(ADDNAME)gcc
LD      = $(BINDIR)/$(ADDNAME)ld
NM      = $(BINDIR)/$(ADDNAME)nm
OBJCOPY = $(BINDIR)/$(ADDNAME)objcopy
OBJDUMP = $(BINDIR)/$(ADDNAME)objdump
RANLIB  = $(BINDIR)/$(ADDNAME)ranlib
STRIP   = $(BINDIR)/$(ADDNAME)strip

PICOTOOL = $(BINDIR)/picotool

OBJS  = vector.o startup.o main.o board.o
OBJS += lib.o serial.o

TARGET = kzload

CFLAGS = -Wall -mcpu=cortex-m33 -mthumb -mfloat-abi=soft -nostdinc -nostdlib -fno-builtin
CFLAGS += -ffreestanding
CFLAGS += -Wno-pointer-sign # The book passes char * strings to unsigned char *
CFLAGS += -I.
#CFLAGS += -g
CFLAGS += -Os
CFLAGS += -DKZLOAD

LFLAGS = -static -T ld.scr -L.

.SUFFIXES: .c .o
.SUFFIXES: .s .o

all :		$(TARGET)

$(TARGET) :	$(OBJS)
		$(CC) $(OBJS) -o $(TARGET) $(CFLAGS) $(LFLAGS)
		cp $(TARGET) $(TARGET).elf
		$(STRIP) $(TARGET)

.c.o :		$<
		$(CC) -c $(CFLAGS) $<

.s.o :		$<
		$(CC) -c $(CFLAGS) $<

# picotool detects the file type from the extension, so convert from .elf
$(TARGET).uf2 :	$(TARGET)
		$(PICOTOOL) uf2 convert $(TARGET).elf $(TARGET).uf2 --family rp2350-arm-s

image :		$(TARGET).uf2

# Hold BOOTSEL while plugging in USB, then run "make write"
write :		$(TARGET).uf2
		$(PICOTOOL) load -x $(TARGET).uf2

clean :
		rm -f $(OBJS) $(TARGET) $(TARGET).elf $(TARGET).uf2
```

### `Makefile`: Book vs Pico 2 H

| | Book (H8) | Pico 2 H |
|---|---|---|
| `PREFIX` | `/usr/local` | `$(shell brew --prefix)` |
| `ARCH` | `h8300-elf` | `arm-none-eabi` |
| Flash tool | `H8WRITE`, `H8WRITE_SERDEV` | `PICOTOOL` |
| `OBJS` | `vector.o startup.o main.o lib.o serial.o` | Adds `board.o` |
| CPU flags | `-mh` | `-mcpu=cortex-m33 -mthumb -mfloat-abi=soft` |
| Extra flags | — | `-ffreestanding`, `-Wno-pointer-sign` |
| `make image` | `kzload.mot` (S-record) | `kzload.uf2` |
| `make write` | h8write over serial | `picotool load` over USB |

- **`PREFIX`:** `brew --prefix` returns `/opt/homebrew` on Apple Silicon and `/usr/local` on Intel Macs, so the Makefile works on both.
- **`-Wno-pointer-sign`:** The book passes string literals (`char *`) to `unsigned char *` parameters. Newer gcc warns about this with `-Wall`, so the warning is disabled to keep the source identical to the book.
- **UF2 conversion:** picotool detects the file type from the extension, so `kzload.uf2` is converted from `kzload.elf` rather than the stripped `kzload`.

## How to Run

```sh
cd 01/bootload
make            # builds kzload and kzload.elf
make image      # builds kzload.uf2
```

1. Hold the BOOTSEL button while plugging in USB. A drive named `RP2350` appears.
2. Drag `kzload.uf2` onto it, or run `make write`.
3. Connect a 3.3V USB-serial adapter:

| Pico 2 H | USB-serial adapter |
|---|---|
| GP0 (pin 1, TX) | RX |
| GP1 (pin 2, RX) | TX |
| GND (pin 3) | GND |

4. Find your adapter with `ls /dev/tty.usb*`, then open a terminal. You should see `Hello World!`.

```sh
minicom -D /dev/tty.usbserial-XXXX -b 115200
```
