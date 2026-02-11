---
applyTo: "**/*.c,**/*.h,**/*.cpp,**/*.hpp,**/*.ld,**/Makefile,**/CMakeLists.txt,**/*.s"
---

# STM32 Target-Specific Instructions

> **Adapt this file to your exact MCU.** Replace the example values below with your actual target, clock configuration, peripheral assignments, and errata. This file is loaded whenever you edit firmware source, build files, or linker scripts.

## MCU Details (EXAMPLE — replace with your target)

- **Part**: STM32F407VGT6
- **Core**: ARM Cortex-M4F with single-precision FPU
- **Max clock**: 168 MHz (HSE 8 MHz → PLL)
- **Flash**: 1 MB (dual-bank), **RAM**: 192 KB (128 KB SRAM + 64 KB CCM)
- **HAL version**: STM32Cube_FW_F4 V1.28.0
- **Toolchain**: `arm-none-eabi-gcc` 13.x, Newlib-nano

## Clock Configuration (EXAMPLE)

```
HSE (8 MHz) → PLL_M=8, PLL_N=336, PLL_P=2, PLL_Q=7
  → SYSCLK = 168 MHz
  → AHB    = 168 MHz (prescaler 1)
  → APB1   =  42 MHz (prescaler 4)  — TIM2-7,12-14 timers run at 84 MHz (×2)
  → APB2   =  84 MHz (prescaler 2)  — TIM1,8-11 timers run at 168 MHz (×2)
  → 48 MHz USB clock from PLL_Q
```

When configuring prescalers and timer frequencies, always calculate from the correct APB timer clock (2× APB when APB prescaler > 1), not from SYSCLK directly.

## Peripheral Allocation (EXAMPLE — document your assignments)

| Peripheral | Usage           | Pins         | Config Notes                    |
|------------|-----------------|--------------|---------------------------------|
| USART2     | Debug console   | PA2 (TX) / PA3 (RX) | 115200 8N1, DMA1 Stream6/Ch4 (TX), Stream5/Ch4 (RX) |
| SPI1       | IMU (MPU6500)   | PA5 (SCK) / PA6 (MISO) / PA7 (MOSI) / PA4 (CS) | 10 MHz, CPOL=0 CPHA=0 |
| I2C1       | EEPROM (AT24C256) | PB6 (SCL) / PB7 (SDA) | 400 kHz Fast Mode |
| TIM2       | Motor PWM       | PA0 (CH1)    | 20 kHz, 16-bit resolution       |
| TIM3       | Encoder input   | PB4 (CH1) / PB5 (CH2) | Encoder mode TI1+TI2 |
| ADC1       | Battery voltage | PA1 (CH1)    | 12-bit, single conversion, DMA  |
| IWDG       | Watchdog        | —            | 1 s timeout                      |
| DMA1 St6   | USART2 TX       | —            | Channel 4, Mem→Periph, byte     |
| DMA1 St5   | USART2 RX       | —            | Channel 4, Periph→Mem, circular |

## DMA Stream/Channel Map (EXAMPLE)

Document all DMA assignments to prevent conflicts. Two peripherals cannot share the same DMA stream.

```
DMA1:
  Stream 5, Channel 4 → USART2_RX
  Stream 6, Channel 4 → USART2_TX
DMA2:
  Stream 0, Channel 0 → ADC1
```

## Linker Script Notes

- Stack size: `_Min_Stack_Size = 0x2000;` (8 KB) — adequate for nested ISRs up to 3 levels
- Heap size: `_Min_Heap_Size = 0x0;` (0 bytes) — enforced no-heap policy; `_sbrk()` triggers a hard fault if called
- CCM RAM (64 KB at 0x10000000): used for performance-critical buffers (DMA cannot access CCM on F4)
- Place frequently accessed lookup tables in CCM to reduce flash wait states

## Known Errata (EXAMPLE — check your silicon revision)

Consult the errata sheet for your exact part and revision. Common examples:

- **ES0206 §2.1.3 (F407)**: In some conditions, ADC DMA requests may not be cleared properly. Workaround: use circular DMA mode for ADC.
- **ES0206 §2.5.1 (F407)**: I2C analog filter may cause false bus busy detection. Workaround: disable analog filter (`hi2c.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE`; toggle SDA/SCL GPIOs before init).
- **Check errata for your specific silicon revision** at https://www.st.com/en/microcontrollers-microprocessors/stm32f407vg.html#documentation

## Build Commands (EXAMPLE — replace with your actual commands)

```bash
# STM32CubeIDE project
# Build debug configuration
make -j$(nproc) -C build/Debug

# CMake-based project
cmake --preset stm32f4-debug
cmake --build --preset stm32f4-debug

# PlatformIO
pio run -e stm32f407_disco

# Flash via ST-Link
st-flash write build/firmware.bin 0x08000000

# Flash via OpenOCD
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg -c "program build/firmware.elf verify reset exit"
```

## Debugging Tips

- Use SWO/ITM trace for non-intrusive printf-style debugging:
  ```c
  // ITM_SendChar is available via CMSIS when SWO is configured
  int _write(int file, char *ptr, int len) {
      for (int i = 0; i < len; i++) ITM_SendChar(*ptr++);
      return len;
  }
  ```
- Use `arm-none-eabi-objdump -d -C <elf>` to inspect generated assembly for performance-critical sections.
- Use `arm-none-eabi-size <elf>` to monitor flash/RAM usage after each change.
- Use `arm-none-eabi-nm --size-sort <elf>` to find the largest symbols.
