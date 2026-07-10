# Blinky

Toggle an onboard LED with a blocking delay loop.

## Setup

1. **CubeMX project created against the STM32F407G-DISC1 board**. Selecting the board — rather than just the MCU — pulls in a default pin configuration for everything physically wired on the Discovery board: the four user LEDs (LD3–LD6) on GPIOD, the user button (B1), the onboard LIS3DSH accelerometer on SPI1, the CS43L22 audio DAC on I2S3, and the USB OTG FS port with Host mode.
2. That's why `MX_I2C1_Init`, `MX_SPI1_Init`, and `MX_I2S3_Init` all appear in `main.c` even though this project doesn't use any of them — they're inherited from the board template, not something added deliberately for a blink demo.
3. **Toolchain**: STM32CubeIDE project, generated code from standalone CubeMX 2.2.0.

## What the generated code does

**`SystemClock_Config()`** — sets up the clock tree from the 8 MHz HSE crystal through the PLL:
- `PLLM = 8` → divides HSE down to a 1 MHz PLL input
- `PLLN = 336` → multiplies to 336 MHz VCO
- `PLLP = 2` → divides down to **168 MHz SYSCLK**
- APB1 prescaler `/4`, APB2 prescaler `/2` (APB1 peripherals are limited to 42 MHz max on the F407, so this is required)

**`MX_GPIO_Init()`** — configures GPIOD pins 12–15 (the four onboard LEDs) as push-pull outputs, plus the pins needed for the other board peripherals mentioned above (their electrical setup, not their actual use).

**`main()` loop** — the only actual "application" logic in the project:
```c
HAL_GPIO_WritePin(GPIOD, GPIO_PIN_13, 1);   // LD3 (orange) on
HAL_Delay(1000);
HAL_GPIO_WritePin(GPIOD, GPIO_PIN_13, 0);   // LD3 off
HAL_Delay(1000);
```
This drives **PD13 (LD3, orange)** high and low with a 1-second blocking delay between each, using `HAL_Delay()` which is a busy-wait built on the SysTick interrupt.
