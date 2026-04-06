# Custom Linker & Startup File from Scratch

Most embedded developers reach for a HAL, a generated startup file, or a board support package on day one. This project does none of that. The goal was to build a working STM32 firmware from the absolute bottom — custom linker script, hand-written startup code, and a `main()` that runs because *we* set it up, not because a code generator did.

## Why bother?

Understanding what happens between reset and `main()` is not academic. When your firmware crashes without a stack trace, when your `.bss` isn't zeroed, when your interrupt vector is in the wrong place — this knowledge is what separates a debug session that takes 10 minutes from one that takes 3 days.

## The memory map

The STM32F4 has a defined memory layout. We need to place code, read-only data, and initialised variables in the right sections.

| Section | Location | Description |
|---------|----------|-------------|
| `.text` | Flash | Code and constants |
| `.rodata` | Flash | Read-only data |
| `.data` | RAM (loaded from Flash) | Initialised variables |
| `.bss` | RAM | Zero-initialised variables |
| `Stack` | RAM (top) | Call stack |

## The linker script

```ld
MEMORY
{
  FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS
{
  .text :
  {
    KEEP(*(.isr_vector))
    *(.text*)
    *(.rodata*)
  } > FLASH

  _sidata = LOADADDR(.data);

  .data :
  {
    _sdata = .;
    *(.data*)
    _edata = .;
  } > RAM AT > FLASH

  .bss :
  {
    _sbss = .;
    *(.bss*)
    *(COMMON)
    _ebss = .;
  } > RAM

  _estack = ORIGIN(RAM) + LENGTH(RAM);
}
```

The key detail: `.data` lives in Flash (as a load address) but runs from RAM. The startup code copies it. `.bss` doesn't need copying — it just needs zeroing. Both are handled in `startup.c`.

## The startup file

```c
extern uint32_t _sdata, _edata, _sidata;
extern uint32_t _sbss,  _ebss;
extern uint32_t _estack;

void Reset_Handler(void) {
    /* Copy .data from Flash to RAM */
    uint32_t *src = &_sidata;
    uint32_t *dst = &_sdata;
    while (dst < &_edata) {
        *dst++ = *src++;
    }

    /* Zero .bss */
    dst = &_sbss;
    while (dst < &_ebss) {
        *dst++ = 0;
    }

    /* Call main */
    main();
}
```

No `SystemInit()`. No HAL. Just memory initialisation and a jump to `main()`.

## Images

Drop your schematic screenshots or scope captures next to the markdown file and reference them like this:

![Linker map output showing section placement](posts/custom-linker-startup/linker_map.png)

The image will render centered with a subtle border inside the post.

## Results

- Binary size: **2.4 KB** for a blinking LED with full interrupt vector
- Zero dependency on any vendor library
- Full visibility into every byte of the memory map
- The kind of understanding that makes every future debugging session faster

## Takeaways

> If you can write a startup file and a linker script from scratch, you understand your hardware. Everything else is just software on top.

The repository includes the full source, Makefile, and OpenOCD flash script. No IDE required.
