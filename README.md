# Embedded Systems — Flight Controller Development

Practical embedded systems development focused on STM32 
microcontrollers, real-time firmware, and custom PCB design. 
This repository documents a structured learning path toward 
designing and implementing a custom drone flight controller 
from scratch — covering firmware, hardware design, and 
control systems implementation.

## Roadmap

### Phase 0 — Simulation (STM32 Nucleo C031C6 on Wokwi)
- [ ] GPIO and timer-based LED control
- [ ] UART serial communication
- [ ] Hardware timer interrupts
- [ ] I2C communication with MPU-6050 IMU
- [ ] PWM signal generation

### Phase 1 — Real Hardware (STM32 Blue Pill)
- [ ] STM32 flashing via ST-Link
- [ ] Real IMU data acquisition and noise analysis
- [ ] Complementary filter for angle estimation
- [ ] PID controller implementation in C
- [ ] DShot ESC protocol

### Phase 2 — System Integration
- [ ] 1-axis stabilization system
- [ ] Custom PCB design in EasyEDA
- [ ] Power distribution board design

### Phase 3 — Custom Flight Controller PCB
- [ ] STM32F405-based FC schematic
- [ ] PCB fabrication and assembly
- [ ] Betaflight target configuration

## Technical Stack
- **MCU:** STM32F1 / STM32F4 series
- **IDE:** STM32CubeIDE, Wokwi simulator
- **PCB:** EasyEDA, JLCPCB fabrication
- **Firmware:** C, HAL, Betaflight
- **Protocols:** UART, I2C, SPI, DShot, PWM

## Background
Final year EEE undergraduate with focus on embedded 
systems, control theory, and hardware design. 
This work supports ongoing preparation for postgraduate 
study in embedded systems engineering.
