# Embedded Systems Learning Guide
## Theory + Video Knowledge Map (Phase 0 → Phase 3)

---

## How to Use This Document

Each phase has two columns of knowledge:
- **Theory** — concepts to understand before coding. Read this first.
- **Video** — what to look for and learn from tutorial videos.

The rule is always: **Read theory → Watch video → Code immediately → Push to GitHub**

---

---

# PHASE 0 — Simulation (Wokwi, No Hardware)

---

## Topic 1 — What is a Microcontroller?

### Theory to Understand
A microcontroller (MCU) is a complete computer on a single chip containing:
- **CPU** — executes your code instructions
- **Flash memory** — stores your program permanently (survives power off)
- **RAM** — temporary memory used while program runs (lost on power off)
- **Peripherals** — built-in hardware modules (UART, I2C, SPI, Timers, ADC)
- **GPIO pins** — physical legs of the chip you connect wires to

The STM32 family runs at 16–480MHz depending on the chip. Your Nucleo C031C6
runs at up to 48MHz. This is vastly faster than Arduino (16MHz) and is why
STM32 is used in flight controllers.

**Key difference from Arduino:**
Arduino hides everything behind simple functions. STM32 exposes the actual
hardware — you configure real registers, real clocks, real peripherals.
This is harder at first but is how professional embedded engineers work.

### What to Learn from Video
- What the STM32 chip physically looks like on a Nucleo board
- How to open STM32CubeIDE and create a new project
- What the .ioc file is (graphical peripheral configuration tool)
- How to select your MCU in CubeIDE
- What "Generate Code" does and why CubeIDE writes some code for you
- Where the main.c file is and where to write your code
- How to build and flash (understand the concept even if using Wokwi)

**Search:** `STM32 CubeIDE getting started tutorial beginners`
**Channel:** Controllerstech — "Getting started with STM32"

---

## Topic 2 — GPIO (General Purpose Input Output)

### Theory to Understand
Every physical pin on the MCU can be configured as either input or output
in software. This is the most basic operation in embedded systems.

**Output mode:**
- Your code controls the voltage on the pin
- HIGH = 3.3V, LOW = 0V
- Use case: turning LED on/off, sending signals

**Input mode:**
- Pin reads external voltage
- Returns HIGH if voltage present, LOW if not
- Use case: reading button press, reading sensor digital output

**LED circuit:**
```
STM32 pin → 330Ω resistor → LED anode → LED cathode → GND

Pin HIGH (3.3V) → current flows → LED lights up
Pin LOW  (0V)   → no current   → LED off
```

**Pull-up and Pull-down resistors:**
When a pin is in input mode and nothing is connected, it floats — reads
random values. Pull-up/pull-down resistors fix the default state:
- Pull-up: pin defaults to HIGH when nothing connected
- Pull-down: pin defaults to LOW when nothing connected
STM32 has these built into the chip — you enable them in software.

**HAL functions you will use:**
```c
HAL_GPIO_WritePin(GPIOx, GPIO_PIN_x, GPIO_PIN_SET);   // set HIGH
HAL_GPIO_WritePin(GPIOx, GPIO_PIN_x, GPIO_PIN_RESET); // set LOW
HAL_GPIO_TogglePin(GPIOx, GPIO_PIN_x);                // flip state
HAL_GPIO_ReadPin(GPIOx, GPIO_PIN_x);                  // read input
```

### What to Learn from Video
- How to configure a pin as output in the .ioc file
- How to configure a pin as input with pull-up enabled
- What GPIOx means (GPIOA, GPIOB, GPIOC — different pin banks)
- How to find which pin number corresponds to which physical leg
- How to use HAL_GPIO_WritePin and HAL_GPIO_TogglePin in code
- How to read a button state with HAL_GPIO_ReadPin

**Search:** `STM32 GPIO LED HAL CubeIDE tutorial`
**Channel:** Controllerstech — "GPIO output STM32"

---

## Topic 3 — Clock System

### Theory to Understand
Everything inside the MCU runs synchronized to a clock signal. Understanding
clocks is essential — wrong clock configuration causes timers to run at wrong
speeds, UART to send garbage data, and SPI to fail silently.

**Clock sources:**
- **HSI** — High Speed Internal oscillator, ~16MHz, built into chip, always available, slightly inaccurate
- **HSE** — High Speed External crystal, more accurate, needs external component
- **PLL** — Phase Locked Loop, multiplies clock frequency (e.g. 16MHz × 3 = 48MHz)
- **LSI/LSE** — Low Speed clocks for watchdog and RTC (not needed now)

**Clock tree concept:**
```
Clock source (HSI/HSE)
        ↓
      PLL (optional multiplier)
        ↓
   SYSCLK (system clock)
     ↓         ↓
  HCLK       PCLK
(CPU, DMA)  (Peripherals: UART, SPI, I2C, Timers)
```

**Why this matters for you:**
When you set up a timer to interrupt every 1ms, the calculation depends
entirely on what clock speed the timer is running at. If you assume 48MHz
but the clock is actually 16MHz your timing will be completely wrong.

**Always verify your clock configuration before doing any timing code.**
CubeIDE has a graphical clock tree view — use it.

### What to Learn from Video
- How to open the Clock Configuration tab in CubeIDE .ioc file
- How to set SYSCLK to maximum for your chip
- What HCLK and PCLK1/PCLK2 are
- How to read what speed your timers are running at
- Why clock speed matters for UART baud rate calculation

**Search:** `STM32 clock configuration CubeIDE tutorial`
**Channel:** Controllerstech — "STM32 clock configuration"

---

## Topic 4 — Polling vs Interrupts

### Theory to Understand
This is one of the most fundamental concepts in embedded systems and directly
relates to how flight controllers achieve 8kHz loop times.

**Polling (blocking):**
```c
while(1) {
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    HAL_Delay(500); // CPU is STUCK here doing nothing
                    // Cannot do anything else during this time
}
```
The CPU is like a person standing at a door constantly asking
"did anyone knock?" — unable to do any other work.

**Interrupt (non-blocking):**
```
Normal code running...
         ↓
Hardware event occurs (timer overflow, button press, data received)
         ↓
CPU immediately pauses current code
         ↓
Jumps to ISR (Interrupt Service Routine) — your handler function
         ↓
Finishes ISR, returns to exactly where it left off
```
The CPU is like a person working at a desk — a doorbell rings,
they answer the door, come back and continue exactly where they stopped.

**ISR (Interrupt Service Routine) rules:**
- Keep ISRs as short and fast as possible
- Never use HAL_Delay inside an ISR
- Never do heavy computation inside an ISR
- Use a flag variable — set it in ISR, handle it in main loop

```c
volatile uint8_t flag = 0; // volatile tells compiler this can change anytime

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM2) {
        flag = 1; // just set a flag, don't do heavy work here
    }
}

while(1) {
    if (flag) {
        flag = 0;
        // do your actual work here
    }
}
```

**Why this matters for drones:**
Betaflight reads the gyro 8000 times per second using a timer interrupt.
The CPU is free to do PID calculations, handle UART, and output motor
signals all between those gyro reads. Without interrupts this is impossible.

### What to Learn from Video
- What NVIC is (Nested Vector Interrupt Controller)
- How to enable an interrupt in CubeIDE .ioc file
- What priority levels mean for interrupts
- How the HAL callback functions work
- How to use volatile variables with interrupts

**Search:** `STM32 interrupts HAL tutorial CubeIDE`
**Channel:** Controllerstech — "External interrupt STM32"

---

## Topic 5 — Timers

### Theory to Understand
A timer is a hardware counter built into the chip. It counts clock pulses
automatically in the background without using CPU time. When it reaches
a configured value it can trigger an interrupt, generate PWM, or both.

**Two key settings:**
- **Prescaler (PSC)** — divides the input clock before it reaches the timer counter
- **Auto-Reload Register (ARR)** — value at which timer resets and triggers interrupt

**Timer frequency calculation:**
```
Timer clock = PCLK / (Prescaler + 1)
Interrupt frequency = Timer clock / (ARR + 1)

Example — interrupt every 1ms on 48MHz clock:
Prescaler = 47    → Timer clock = 48MHz / 48 = 1MHz
ARR = 999         → Interrupt = 1MHz / 1000 = 1000Hz = every 1ms
```

**Types of timer modes:**
- **Basic timer** — just counts and interrupts, no output pin
- **PWM mode** — generates PWM signal on output pin
- **Input capture** — measures pulse width on input pin
- **Output compare** — triggers action when counter matches value

**HAL functions for timers:**
```c
HAL_TIM_Base_Start_IT(&htim2);  // start timer with interrupt enabled
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1); // start PWM output

// Callback — called automatically when timer overflows
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM2) {
        // your code here — runs every timer period
    }
}
```

**Why this matters for drones:**
The main flight controller loop runs inside a timer interrupt callback.
Gyro is read, PID is calculated, motor outputs are updated — all inside
one timer callback running 8000 times per second.

### What to Learn from Video
- How to configure a timer in CubeIDE .ioc file
- How to calculate prescaler and ARR for a desired frequency
- How to enable the timer update interrupt in NVIC
- How HAL_TIM_Base_Start_IT works
- How to use HAL_TIM_PeriodElapsedCallback
- How to configure timer for PWM output (you'll use this later)

**Search:** `STM32 timer interrupt HAL CubeIDE tutorial`
**Channel:** Controllerstech — "Timer interrupt STM32"

---

## Topic 6 — UART Communication

### Theory to Understand
UART (Universal Asynchronous Receiver Transmitter) is the simplest serial
communication protocol. It uses just two wires — TX and RX.

**How it works:**
```
Device A TX ──────────────→ Device B RX
Device A RX ←────────────── Device B TX
GND ────────────────────────────────── GND (always connect grounds)
```

**Key settings (must match on both devices):**
- **Baud rate** — speed in bits per second (typical: 9600, 115200, 921600)
- **Data bits** — usually 8
- **Stop bits** — usually 1
- **Parity** — usually None

**In drones UART is used for:**
- Printing debug messages to PC during development
- RC receiver sending stick commands to FC (CRSF/ELRS protocol)
- ESC telemetry (RPM, temperature, current) coming back to FC
- GPS sending position data to FC
- Betaflight Configurator talking to FC over USB (which is UART internally)

**HAL functions:**
```c
// Transmit — send data
HAL_UART_Transmit(&huart2, (uint8_t*)"Hello\r\n", 7, HAL_MAX_DELAY);

// Receive — blocking, waits for data
HAL_UART_Receive(&huart2, buffer, length, timeout);

// Printf over UART (needs retargeting)
printf("Gyro X: %d\r\n", gyro_x);
```

**Transmit vs Receive modes:**
- Blocking: function waits until complete (simple but CPU is stuck)
- Interrupt: function returns immediately, callback fires when done
- DMA: hardware handles transfer, CPU completely free (best for high speed)

### What to Learn from Video
- How to configure UART in CubeIDE .ioc file
- How to set baud rate, data bits, stop bits
- How to use HAL_UART_Transmit to send a string
- How to redirect printf to UART (retargeting)
- How to use UART receive interrupt
- How to view output on a serial monitor (PC side)

**Search:** `STM32 UART transmit HAL CubeIDE tutorial`
**Channel:** Controllerstech — "UART STM32 transmit receive"

---

## Topic 7 — I2C Protocol

### Theory to Understand
I2C (Inter-Integrated Circuit) uses only 2 wires to connect multiple
devices — SDA (data) and SCL (clock).

**How it works:**
```
MCU (Master)
    |
SDA ──┬──────────── Device 1 (address 0x68) MPU-6050
SCL ──┤──────────── Device 2 (address 0x76) Barometer
      └──────────── Device 3 (address 0x3C) OLED display

Pull-up resistors (4.7kΩ) required on both SDA and SCL to 3.3V
```

**Every I2C device has a unique address** (7-bit number, e.g. 0x68).
The master (your STM32) calls this address before sending data —
only the device with that address responds.

**I2C transaction structure:**
```
START → Device Address + Write → Register Address → Data → STOP
START → Device Address + Read  → Read Data → STOP
```

**Reading a sensor register:**
```c
// Write: tell sensor which register to read
HAL_I2C_Master_Transmit(&hi2c1, 0x68<<1, &reg_addr, 1, HAL_MAX_DELAY);

// Read: get the data from that register
HAL_I2C_Master_Receive(&hi2c1, 0x68<<1, &data, 1, HAL_MAX_DELAY);

// Or combined with Mem functions (easier):
HAL_I2C_Mem_Read(&hi2c1, 0x68<<1, reg_addr, 1, &data, 1, HAL_MAX_DELAY);
```

**MPU-6050 specific:**
- I2C address: 0x68 (AD0 pin LOW) or 0x69 (AD0 pin HIGH)
- Gyroscope registers: 0x43, 0x45, 0x47 (X, Y, Z high bytes)
- Accelerometer registers: 0x3B, 0x3D, 0x3F (X, Y, Z high bytes)
- Power management register: 0x6B (must write 0x00 to wake up sensor)
- WHO_AM_I register: 0x75 (should return 0x68, used to verify connection)

**Why I2C for MPU-6050:**
Most breakout boards use I2C for simplicity. Real flight controllers use
SPI for the IMU because SPI is 10x faster — allowing 8kHz gyro reads.
But for learning, I2C is perfect.

### What to Learn from Video
- How to configure I2C in CubeIDE .ioc file
- What clock speed to set (100kHz standard, 400kHz fast mode)
- How address shifting works (why 0x68 becomes 0x68<<1)
- How to use HAL_I2C_Mem_Read and HAL_I2C_Mem_Write
- How to wake up MPU-6050 (write to power register)
- How to read and convert raw gyro values to degrees/second
- How to verify sensor connection using WHO_AM_I register

**Search:** `STM32 MPU6050 I2C HAL CubeIDE tutorial`
**Channel:** Controllerstech — "MPU6050 STM32 I2C"

---

## Topic 8 — PWM (Pulse Width Modulation)

### Theory to Understand
PWM allows a digital pin (only HIGH or LOW) to simulate analog output
by switching rapidly between HIGH and LOW at a fixed frequency.

**Duty cycle:**
```
100% duty → ████████████ always HIGH → full brightness/speed
 75% duty → █████████___ 75% HIGH    → 75% brightness/speed
 50% duty → ██████______ 50% HIGH    → half brightness/speed
 25% duty → ███_________ 25% HIGH    → 25% brightness/speed
  0% duty → ____________ always LOW  → off
```

**Two key parameters:**
- **Frequency** — how many cycles per second (Hz)
- **Duty cycle** — percentage of time the signal is HIGH

**PWM for ESC motor control:**
```
Traditional PWM (old standard):
- Frequency: 50Hz (one pulse every 20ms)
- 1000μs pulse = 0% throttle (motor off)
- 1500μs pulse = 50% throttle
- 2000μs pulse = 100% throttle (full speed)

DShot (digital, modern):
- No analog timing — digital packet
- Much more precise, no calibration needed
- Supports bidirectional telemetry
- DShot300, DShot600 — number = speed in kbits/s
```

**HAL PWM setup:**
```c
// Start PWM on timer 3, channel 1
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);

// Change duty cycle — CCR value from 0 to ARR
// If ARR = 999, then CCR = 500 = 50% duty cycle
__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 500);
```

**Complementary filter preview (Phase 1):**
You will use gyro + accelerometer data together to estimate angle:
```
angle = 0.98 × (angle + gyro_rate × dt) + 0.02 × accel_angle

Gyro:  accurate short term, drifts over time
Accel: noisy short term, accurate long term
Filter: combines both — best of each
```

### What to Learn from Video
- How to configure a timer for PWM output in CubeIDE
- How to select the correct timer channel and output pin
- How ARR and CCR values relate to frequency and duty cycle
- How to use HAL_TIM_PWM_Start
- How to use __HAL_TIM_SET_COMPARE to change duty cycle at runtime
- How to connect an LED or servo to test PWM output

**Search:** `STM32 PWM timer HAL CubeIDE tutorial`
**Channel:** Controllerstech — "PWM STM32 CubeIDE"

---

---

# PHASE 1 — Real Hardware (STM32 Blue Pill)

---

## Topic 9 — STM32 Toolchain (Real Hardware)

### Theory to Understand
Moving from Wokwi to real hardware requires a programmer/debugger.
The Blue Pill cannot program itself — you need an ST-Link V2.

**ST-Link connection to Blue Pill:**
```
ST-Link    →    Blue Pill
SWDIO      →    DIO
SWCLK      →    DCLK
GND        →    GND
3.3V       →    3.3V (power from ST-Link)
```

**SWD (Serial Wire Debug):**
SWD is a 2-wire debug protocol. Unlike UART which just transfers data,
SWD lets you: flash firmware, set breakpoints, read memory live,
step through code instruction by instruction. This is how you debug.

**Important Blue Pill note:**
Many cheap Blue Pills come with a counterfeit STM32 chip (CKS32 or CS32).
These work for basic code but may fail on some HAL functions.
If something doesn't work as expected, the chip may be counterfeit.

### What to Learn from Video
- How to wire ST-Link V2 to Blue Pill correctly
- How to configure CubeIDE debug settings for Blue Pill
- How to flash firmware and verify it ran
- How to use the debugger — breakpoints, watch variables, step through code
- How to use a serial monitor (RealTerm or PuTTY) to see UART output

**Search:** `STM32 Blue Pill ST-Link CubeIDE flash tutorial`

---

## Topic 10 — Real Sensor Noise and Filtering

### Theory to Understand
This is where simulation and reality diverge completely. Real sensors
are noisy — the MPU-6050 readings will never be perfectly stable.

**Sources of noise:**
- Electrical noise from power supply
- Vibration from motors coupling into the IMU
- Temperature effects on sensor output
- Quantization noise from ADC inside the sensor

**Raw gyro data example:**
```
Wokwi (simulation):  0, 0, 0, 0, 0, 0, 0  ← perfectly stable
Real hardware:       2, -1, 3, 0, -2, 1, 3 ← constantly jittering
With motors running: 15, -8, 22, -11, 18   ← much worse
```

**Complementary filter — angle estimation:**
```c
// dt = time since last update in seconds
// gyro_rate = degrees per second from gyro
// accel_angle = angle calculated from accelerometer

float angle = 0.98f * (angle + gyro_rate * dt)
            + 0.02f * accel_angle;

// 0.98 and 0.02 are filter coefficients — must sum to 1.0
// Adjust these based on your noise levels
```

**Low pass filter — simple noise reduction:**
```c
// Alpha between 0 and 1
// Lower alpha = more smoothing, more lag
// Higher alpha = less smoothing, less lag
filtered = alpha * new_value + (1 - alpha) * filtered;
```

### What to Learn from Video
- How to plot serial data in real time (Arduino Serial Plotter works)
- How to observe noise visually on a graph
- How to implement and tune a complementary filter
- How to calculate dt (delta time) accurately using HAL_GetTick()
- How accelerometer angle calculation works (atan2 function)

**Search:** `STM32 MPU6050 complementary filter angle estimation`

---

## Topic 11 — PID Controller Implementation

### Theory to Understand
PID (Proportional Integral Derivative) is the core algorithm of every
flight controller. You studied this in EEE control systems — now you
implement it in C on real hardware.

**The three terms:**
```
Error = Setpoint - Measured_Value
      = desired_angle - current_angle

P (Proportional): output = Kp × error
  → Reacts to current error
  → Too high: oscillates
  → Too low: slow response

I (Integral): output = Ki × sum_of_error × dt
  → Reacts to accumulated error over time
  → Eliminates steady-state error
  → Too high: windup, instability

D (Derivative): output = Kd × (error - last_error) / dt
  → Reacts to rate of change of error
  → Dampens oscillation
  → Too high: amplifies noise
```

**PID implementation in C:**
```c
typedef struct {
    float Kp, Ki, Kd;
    float integral;
    float last_error;
    float output;
} PID_t;

float PID_Calculate(PID_t *pid, float setpoint,
                    float measured, float dt) {
    float error = setpoint - measured;

    pid->integral += error * dt;

    // Anti-windup — clamp integral
    if (pid->integral > 100.0f)  pid->integral = 100.0f;
    if (pid->integral < -100.0f) pid->integral = -100.0f;

    float derivative = (error - pid->last_error) / dt;
    pid->last_error = error;

    pid->output = pid->Kp * error
                + pid->Ki * pid->integral
                + pid->Kd * derivative;
    return pid->output;
}
```

**Tuning procedure:**
1. Set Ki = 0, Kd = 0, increase Kp until oscillation starts
2. Reduce Kp slightly below oscillation point
3. Increase Kd to dampen oscillation
4. Add small Ki to eliminate steady-state error

### What to Learn from Video
- How to structure a PID in C as a reusable function or struct
- How to calculate dt reliably using timer or HAL_GetTick
- How to implement anti-windup for the integral term
- How to log PID output over UART and plot it
- How to tune PID gains by observing the plotted response

**Search:** `PID controller STM32 C implementation tutorial`

---

## Topic 12 — DShot ESC Protocol

### Theory to Understand
DShot is a digital motor control protocol replacing traditional analog PWM.
Instead of pulse width, it sends a precise digital packet.

**Why DShot over PWM:**
```
Traditional PWM problems:
- Timing sensitive — 1μs error = noticeable throttle jump
- Needs calibration (min/max endpoints)
- No feedback from ESC

DShot advantages:
- Digital — no timing errors
- No calibration needed
- Bidirectional variant returns RPM, voltage, temperature
```

**DShot packet structure:**
```
16-bit frame:
[11 bits throttle value][1 bit telemetry request][4 bits CRC]

Throttle range: 0 = disarmed, 48–2047 = 0–100% throttle
```

**DShot variants by speed:**
```
DShot150  →  150 kbits/s  (slowest)
DShot300  →  300 kbits/s  (common)
DShot600  →  600 kbits/s  (fast)
DShot1200 → 1200 kbits/s  (fastest)
```

**Implementation uses STM32 timer in PWM mode:**
Each bit is sent as a PWM pulse — bit 1 is a long pulse,
bit 0 is a short pulse. The timer generates these precisely.
DMA is used to stream the 16 bits without CPU involvement.

### What to Learn from Video
- How DShot bit timing works (bit period, 0 vs 1 pulse widths)
- How to implement DShot using STM32 timer + DMA
- How to arm an ESC with DShot (send 0 throttle repeatedly)
- How to read back telemetry on bidirectional DShot
- How to test with a real motor safely (secure it to a surface)

**Search:** `STM32 DShot ESC protocol implementation`

---

---

# PHASE 2 — System Integration

---

## Topic 13 — Power Systems and LiPo Batteries

### Theory to Understand
LiPo (Lithium Polymer) batteries are the standard power source for drones.
They are high energy density but require careful handling.

**LiPo cell voltage:**
```
Fully charged: 4.20V per cell
Nominal:       3.70V per cell
Minimum:       3.50V per cell (never go below this)
Destroyed:     below 3.0V per cell

3S battery = 3 cells in series
3S fully charged: 4.20 × 3 = 12.6V
3S nominal:       3.70 × 3 = 11.1V
3S minimum:       3.50 × 3 = 10.5V
```

**BEC (Battery Eliminator Circuit):**
Your STM32 runs at 3.3V. Your battery is 11.1V.
A BEC steps down the voltage:
```
LiPo (11.1V) → BEC → 5V or 3.3V → STM32 and sensors
```
Linear BEC: simple, wastes excess voltage as heat
Switching BEC: efficient, used on flight controllers

**Voltage regulator on Blue Pill:**
The Blue Pill has an onboard 3.3V linear regulator (AMS1117).
It accepts up to 5V on the 5V pin.
Do NOT connect LiPo directly to Blue Pill — use a BEC first.

**Low voltage cutoff in code:**
```c
float battery_voltage = read_adc() * voltage_divider_factor;
if (battery_voltage < 10.5f) {  // 3S minimum
    disarm_motors();
    // signal low battery warning
}
```

### What to Learn from Video
- LiPo charging safety — always use a balance charger
- Never leave LiPo charging unattended
- LiPo storage voltage (3.8V per cell)
- How to use a LiPo alarm/buzzer for low voltage warning
- How to measure battery voltage with STM32 ADC through voltage divider

**Search:** `LiPo battery safety guide FPV drone`
**Search:** `STM32 ADC voltage measurement tutorial`

---

## Topic 14 — PCB Design Fundamentals (EasyEDA)

### Theory to Understand
A PCB (Printed Circuit Board) is a board with copper traces that connect
components electrically — replacing the jumper wires of your breadboard.

**PCB layers:**
```
Simple 2-layer PCB (what you'll start with):
Top copper layer    → signal traces, component pads
FR4 substrate       → the physical board material (fiberglass)
Bottom copper layer → ground plane, additional traces

Advanced 4-layer PCB (flight controllers):
Layer 1 → signal traces
Layer 2 → ground plane (solid copper — reduces noise)
Layer 3 → power plane (3.3V, 5V, VBAT)
Layer 4 → signal traces
```

**Ground plane:**
A solid copper layer connected entirely to GND. Critical for:
- Reducing electromagnetic interference (EMI)
- Providing a low-impedance return path for all currents
- Shielding sensitive signals (like IMU) from noisy power traces

**Trace width for current:**
```
Thin trace (0.2mm) → low current signals (SPI, I2C, UART)
Medium trace (0.5mm) → moderate current (3.3V power rail)
Wide trace (1-2mm+)  → high current (motor power, battery)

Rule: wider trace = less resistance = less heat = less voltage drop
```

**Decoupling capacitors:**
Place small capacitors (100nF) close to every IC power pin.
They filter out high-frequency noise on the power supply.
Without them, ICs can reset randomly or produce noisy output.

**EasyEDA workflow:**
```
1. Draw schematic   → connect components logically
        ↓
2. Assign footprints → match component to physical size
        ↓
3. Convert to PCB   → components appear on board
        ↓
4. Place components → arrange physically on board
        ↓
5. Route traces     → draw copper connections
        ↓
6. Add ground fill  → flood unused areas with GND copper
        ↓
7. DRC check        → verify no design errors
        ↓
8. Export Gerbers   → manufacturing files
        ↓
9. Order from JLCPCB → receive boards in 2 weeks
```

### What to Learn from Video
- How to start a new project in EasyEDA
- How to place components from the library
- How to draw wire connections in schematic
- How to add power symbols (VCC, GND)
- How to convert schematic to PCB layout
- How to manually route traces
- How to add copper pour (ground fill)
- How to run DRC and fix errors
- How to export Gerber files and order from JLCPCB

**Search:** `EasyEDA tutorial beginners PCB design`
**Search:** `EasyEDA schematic to PCB complete tutorial`

---

---

# PHASE 3 — Custom Flight Controller PCB

---

## Topic 15 — STM32F405 Minimum System

### Theory to Understand
The STM32F405 is the chip used in most serious flight controllers
(SpeedyBee F405, Matek F405, etc.). It runs at 168MHz with FPU
(Floating Point Unit) — essential for fast PID calculations.

**Minimum system components:**
```
STM32F405RGT6 (the chip itself)
       +
8MHz HSE crystal + 2× load capacitors (18-22pF)
       +
100nF + 10μF decoupling caps on every power pin
       +
Boot0 pin → GND (normal boot from flash)
       +
Reset button + 100nF cap
       +
USB connector (for Betaflight Configurator)
       +
3.3V LDO regulator (e.g. SPX3819)
       +
SWD debug header (4 pins: SWDIO, SWCLK, GND, 3.3V)
```

**Why HSE crystal:**
STM32F405 PLL can multiply 8MHz HSE to exactly 168MHz.
Using HSI (internal) gives ~168MHz but less accurate —
acceptable for most things but USB requires exact 48MHz
which only works reliably with HSE.

**SPI vs I2C for IMU on real FC:**
```
I2C MPU-6050 (learning/Phase 1):
- Max speed: 400kHz (fast mode)
- Gyro read at ~400Hz maximum practical rate

SPI ICM-42688 (real FC):
- Max speed: 24MHz
- Gyro read at 8000Hz (8kHz) easily
- This is why all real FCs use SPI for IMU
```

### What to Learn from Video
- How to read STM32F405 datasheet — power pins, decoupling requirements
- How to place crystal circuit correctly in schematic
- How to configure USB in CubeIDE for STM32F405
- How to set up SPI for IMU communication at high speed
- How to configure DMA for SPI — CPU-free gyro reads

**Search:** `STM32F405 minimum system schematic design`
**Search:** `STM32F405 flight controller PCB design tutorial`

---

## Topic 16 — Betaflight Target Configuration

### Theory to Understand
Betaflight is the firmware that runs on most racing/freestyle FC boards.
It is open source C code running on STM32. To run it on your custom board
you need to create a "target" — a configuration file that tells Betaflight
which pins connect to which functions on your specific board.

**Betaflight unified target system:**
```
config.h file defines:
- Which UART is connected to the RC receiver
- Which SPI bus and CS pin the IMU is on
- Which timer outputs drive which motors
- Which ADC pin measures battery voltage
- Which I2C bus connects to barometer
```

**Motor output mapping:**
```
Motor 1 (front right) → TIM3_CH1 → PA6
Motor 2 (rear left)   → TIM3_CH2 → PA7
Motor 3 (front left)  → TIM3_CH3 → PB0
Motor 4 (rear right)  → TIM3_CH4 → PB1
```

**Getting Betaflight running on custom hardware:**
1. Fork Betaflight repository on GitHub
2. Copy an existing similar target as starting point
3. Modify pin definitions to match your board
4. Compile firmware
5. Flash via DFU (USB) or ST-Link
6. Connect Betaflight Configurator — verify sensor data

### What to Learn from Video
- How to set up Betaflight build environment
- How Betaflight target files are structured
- How to compile Betaflight from source
- How to flash via DFU mode
- How to verify gyro data in Betaflight Configurator

**Search:** `Betaflight custom target configuration tutorial`
**Search:** `Betaflight compile from source custom board`

---

---

# Quick Reference — Video Search Cheatsheet

| Phase | Topic | Search Term | Channel |
|-------|-------|-------------|---------|
| 0 | Getting started | `STM32 CubeIDE beginner tutorial` | Controllerstech |
| 0 | GPIO LED | `STM32 GPIO LED HAL CubeIDE` | Controllerstech |
| 0 | Clock config | `STM32 clock configuration CubeIDE` | Controllerstech |
| 0 | Interrupts | `STM32 external interrupt HAL` | Controllerstech |
| 0 | Timers | `STM32 timer interrupt HAL CubeIDE` | Controllerstech |
| 0 | UART | `STM32 UART transmit HAL CubeIDE` | Controllerstech |
| 0 | I2C + MPU6050 | `STM32 MPU6050 I2C HAL CubeIDE` | Controllerstech |
| 0 | PWM | `STM32 PWM timer HAL CubeIDE` | Controllerstech |
| 1 | Flashing hardware | `STM32 Blue Pill ST-Link CubeIDE flash` | Any |
| 1 | Complementary filter | `STM32 MPU6050 complementary filter angle` | Any |
| 1 | PID in C | `PID controller STM32 C implementation` | Any |
| 1 | DShot | `STM32 DShot ESC protocol implementation` | Any |
| 2 | LiPo safety | `LiPo battery safety FPV guide` | Any |
| 2 | EasyEDA basics | `EasyEDA tutorial beginners PCB design` | Any |
| 3 | STM32F405 design | `STM32F405 minimum system schematic` | Any |
| 3 | Betaflight target | `Betaflight custom target configuration` | Any |

---

*Document version 1.0 — created March 2026*
*Update this document as you progress through phases*
