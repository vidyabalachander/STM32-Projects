# PWM (Pulse Width Modulation)

Drives the onboard green LED (PD12) at variable brightness using TIM4 Channel 1 in PWM mode, instead of the hard on/off blink from the Blinky project.

**Pulse Width Modulation (PWM)** is a way to fake an analog output — like a dimmed LED — using only a digital on/off pin. Instead of holding the pin fully high or fully low, you switch it on and off very fast, and vary how long it stays "on" within each cycle. If the switching is fast enough, the LED (or motor, or speaker) responds to the *average* voltage over time rather than each individual pulse, so it looks dimmer or brighter depending on that average.

**Duty cycle** is the percentage of each cycle the signal spends high: `duty = time_high / total_period`. 0% duty is fully off, 100% is fully on, 50% duty looks roughly like half brightness. On this timer, duty cycle is set by the compare value (CCR) relative to the period (ARR) — see the timer math below.

## Setup before writing any code

1. **CubeMX project created against the STM32F407G-DISC1 board**, same as Blinky — so the same inherited peripherals show up (`MX_I2C1_Init`, `MX_SPI1_Init`, `MX_I2S3_Init`), none of them used by this project either.
2. **TIM4 enabled with Channel 1 set to PWM Generation CH1.** CubeMX automatically routes this to **PD12**, since that's the only pin broken out to TIM4_CH1 on this package.
3. Timer parameters set in CubeMX: **Prescaler = 83**, **Period (ARR) = 255**, Counter Mode = Up.
4. **Toolchain**: STM32CubeIDE project, generated code from standalone CubeMX 2.2.0.

## What the generated code does

**`SystemClock_Config()`** — same 168 MHz SYSCLK setup as Blinky (8 MHz HSE → PLL → 168 MHz, APB1 `/4`, APB2 `/2`).

**`MX_TIM4_Init()`** — configures TIM4 as a PWM source on Channel 1:
```c
htim4.Init.Prescaler = 83;
htim4.Init.Period = 255;
sConfigOC.OCMode = TIM_OCMODE_PWM1;
sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
```
`PWM1` mode means the output is high while the counter is below the compare value (CCR), and low once the counter passes it — so the compare value directly controls duty cycle.

**Timer clock math** — TIM4 sits on APB1. Since the APB1 prescaler isn't 1, HAL doubles the timer clock automatically:
```
TIM4 clock = (168 MHz / 4) × 2 = 84 MHz
Counter tick rate = 84 MHz / (83 + 1) = 1 MHz   →  1 µs per tick
PWM frequency = 1 MHz / (255 + 1) ≈ 3.9 kHz
```
~3.9 kHz is well above the flicker-fusion threshold, so the LED reads as a steady dimmed brightness instead of a visible strobe.

**`main()` loop** — the only actual "application" logic:
```c
__HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_1, 255);
HAL_Delay(500);
__HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_1, 100);
HAL_Delay(500);
__HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_1, 30);
HAL_Delay(500);
__HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_1, 0);
HAL_Delay(2000);
```
`__HAL_TIM_SET_COMPARE` writes directly to the CCR1 register, changing duty cycle on the fly. With Period (ARR) = 255, the counter runs through 256 values (0–255) each cycle, so CCR maps onto a 0–255 brightness scale: 255 ≈ full brightness (~99.6% duty, since output is high for CNT = 0–254), 100 ≈ 39%, 30 ≈ 12%, 0 = off. Each level is held for its `HAL_Delay()` before moving to the next.
