# 01 — LED Blink with Timer Interrupt

## What This Does
Blinks an onboard LED using a hardware timer interrupt
on STM32 Nucleo C031C6. Uses HAL timer callback instead
of HAL_Delay so the CPU is not blocked.

## Key Concepts Learned
- Timer prescaler and period calculation
- HAL_TIM_PeriodElapsedCallback interrupt handler
- Difference between polling and interrupt-driven code

## Simulation
Wokwi link will be added upon completion

## Code
See main.c in this folder
