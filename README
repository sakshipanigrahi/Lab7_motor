# Lab 7: Temperature-Limited Motor Speed Controller (TM4C123 / Tiva C)

A beginner-friendly guide to this project. Read it top to bottom once, then use it as a reference.

---

## 1. What does this program do?

It runs a small DC motor and lets you control it three ways:

1. **Potentiometer (the knob)** sets the motor speed, from 0% to 100%.
2. **DHT11 temperature sensor** protects the motor: when the room gets warm, the program automatically slows the motor down, and stops it when it gets too hot.
3. **Switch SW1** (the button on the Tiva board) reverses the motor direction each time you press it.

An **LCD** shows what is going on:

```
ADC:2048 T:25.0C
Duty: 50% FWD
```

- Line 1: the raw knob reading (0-4095) and the temperature.
- Line 2: the motor speed ("duty") in percent and the direction (FWD or REV).

---

## 2. What you need

| Item | Notes |
|---|---|
| Tiva C LaunchPad (TM4C123GH6PM) | Programmed from Code Composer Studio (CCS) |
| Trainer board with LCD | 16x2 LCD wired in 8-bit mode |
| Potentiometer | Gives a voltage from 0 to 3.3 V |
| DHT11 sensor | Temperature sensor (measures 0 to 50 deg C) |
| L298N motor driver + DC motor | Needs its own motor power supply |
| USB cable | Powers and programs the Tiva |

This project does **not** use interrupts, so it uses the **default startup file** (not the modified EduARM/keypad startup).

---

## 3. Wiring (pin map)

| Tiva pin | Goes to | What it does |
|---|---|---|
| PB0 - PB7 | LCD D0 - D7 | LCD data (8 wires) |
| PA6 | LCD RS | Tells LCD "this is a command" (0) or "this is a letter" (1) |
| PA7 | LCD E | "Enable" pulse: tells the LCD to read the data |
| PE3 | Potentiometer middle pin | Analog input (ADC channel AIN0) |
| PE4 | DHT11 DATA pin | Digital input/output (the sensor talks on this one wire) |
| PF2 | L298N ENA | PWM output: controls motor speed |
| PD1 | L298N IN1 | Direction wire 1 |
| PD2 | L298N IN2 | Direction wire 2 |
| PF4 | SW1 (on the board) | Direction-change button (pressed = 0) |

**Do not forget:**
- Connect the **grounds together**: Tiva GND, L298N GND, DHT11 GND, motor supply GND.
- DHT11 needs 3.3 V (or 5 V depending on your module) and GND. The code also turns on a pull-up on PE4 in case your module has no pull-up resistor.
- The potentiometer's two outer pins go to 3.3 V and GND.

---

## 4. How to build and run (CCS), step by step

1. Open **Code Composer Studio**.
2. Create a new CCS project for the **TM4C123GH6PM** (Tiva C Series). Pick the **default startup file** project template (or copy in the default `startup_ccs.c`).
3. Add this `main.c` to the project (delete any other `main.c`).
4. Make sure the header file `inc/tm4c123gh6pm.h` is in the project. The line `#include "inc/tm4c123gh6pm.h"` looks for it inside a folder called `inc`.
5. Plug in the Tiva with USB.
6. Click the **Build** (hammer) icon. Wait for "Build Finished" with **0 errors**.
7. Click the **Debug** (green bug) icon. CCS downloads the program to the board.
8. Click **Resume** (green play button) to run.

**What you should see:**
1. The LCD shows `LAB 7 READY` for about 1 second.
2. Then the two lines of info appear and keep updating.
3. Turn the potentiometer: ADC and Duty change, and the motor speeds up or slows down.
4. Press SW1: the motor stops for a moment, then spins the other way. The LCD shows FWD or REV.
5. Warm the sensor (hold it gently in your hand): the temperature rises and the duty drops.

---

## 5. How the program works (explained simply)

### The big picture

A C program starts at `main()`. Ours does setup once, then goes into `while (1)`, which means "repeat forever". Each trip around the loop takes about 0.1 second.

```
SETUP (once):   gpio_init -> adc_init -> pwm_init -> lcd_init
LOOP (forever): read knob -> read temperature (every 1 s) -> work out speed
                -> check button -> update LCD (every 0.5 s) -> wait 100 ms
```

### The first lines: `#include` and `#define`

```c
#include <stdio.h>      // gives us sprintf (used to build LCD text)
#include <stdint.h>     // gives us uint8_t, uint32_t (numbers of an exact size)
#include "inc/tm4c123gh6pm.h"   // names for the chip's registers
```

- `uint8_t` = a number from 0 to 255 (8 bits).
- `uint32_t` = a big positive number (32 bits).
- A **register** is a special memory box inside the chip. Writing a number to it changes how the hardware behaves. `GPIO_PORTB_DATA_R` is the box for port B's pins.

`#define NAME value` means "wherever you see NAME, use this value". Example: `#define DHT_PIN 0x10`. The `0x` means the number is written in **hexadecimal**. `0x10` is binary `0001 0000`, which is "bit 4" = pin 4 (PE4).

### Delay functions (using SysTick timer)

`delay_us(us)` waits that many **microseconds**. `delay_ms(ms)` waits that many **milliseconds** (it just calls `delay_us(1000)` many times).

The chip runs at 16 MHz, so it counts 16 ticks per microsecond. We load `16 * us - 1` into the SysTick timer and wait until it says "I finished counting".

### LCD functions

- `lcd_write(value, rs)`: puts 8 bits on PB0-PB7, sets RS (command or data), then pulses E high then low so the LCD grabs the data.
- `lcd_cmd(c)`: send a command (for example `0x01` clears the screen, `0x80` moves to line 1, `0xC0` moves to line 2).
- `lcd_data(d)`: send one character to show.
- `lcd_print("text")`: send many characters, one at a time, until the text ends.
- `lcd_init()`: the startup sequence the LCD needs (8-bit mode, 2 lines, display on, cursor off).

### `gpio_init()`: set up the pins

Turns on the clock for ports A, B, D, E, F (`0x3B` = ports A, B, D, E, F), then for each pin chooses:
- **DIR**: 1 = output, 0 = input.
- **DEN**: digital enable (turn the pin "on" as a digital pin).
- **AFSEL / PCTL**: hand the pin over to a special hardware block (PF2 goes to PWM).
- **AMSEL**: analog mode (PE3 for the potentiometer).
- **PUR**: internal pull-up resistor (SW1 and DHT11).

### ADC: reading the potentiometer

`adc_init()` sets up ADC0 sequencer 3 to read channel AIN0 (pin PE3), one sample at a time.
`adc_read()` starts a conversion, waits until it is done, and returns a number from **0 to 4095** (12 bits).
- Knob fully one way = 0, fully the other way = 4095.

### DHT11: reading the temperature

The DHT11 talks over **one wire** using very precise timing:
1. Tiva pulls the wire low for 20 ms ("hello, send data").
2. Sensor answers with a low and a high pulse.
3. Sensor sends **40 bits** (5 bytes): humidity, humidity decimal, temperature, temperature decimal, and a checksum.
4. A short high pulse (about 27 us) means bit 0, a long one (about 70 us) means bit 1. The code waits 40 us then looks: still high = 1, already low = 0.
5. The **checksum** (sum of the first four bytes) is checked. If it does not match, the reading is thrown away.

`dht_read()` returns 1 if OK and 0 if failed. The temperature is stored in **tenths of a degree**: `254` means 25.4 deg C. (We use whole numbers because they are simpler than decimals in C.)

If a read fails, the program keeps the **old** temperature.

### PWM: controlling the motor speed

**PWM** = turning the motor pin on and off very fast (1000 times a second). The longer it stays on in each cycle, the faster the motor goes.

- `PWM_LOAD = 7999` means one cycle is 8000 counts. With an 8 MHz PWM clock (16 MHz divided by 2), that is **1 kHz**.
- `duty` is the "on" percentage: 0 = always off, 50 = half the time on, 100 = almost always on.
- `pwm_set_duty(duty)` converts the percent into a compare value for the hardware.
- The output comes out of **PF2** (module PWM1, generator 3).

### Motor direction

The L298N has two direction inputs. `motor_dir()`:
- FWD: IN1 = 1, IN2 = 0
- REV: IN1 = 0, IN2 = 1

### The main loop, step by step

1. **Read the knob** -> `pot` (0-4095).
2. **Read DHT11 once every 10 loops** (about once a second; the sensor is slow and cannot be read too often).
3. **Duty from knob:** `duty = pot * 100 / 4095` (gives 0-100).
4. **Temperature scaling:** `scale` is 100% when it is cool, falls in a straight line, and reaches 0% when it is hot (see table below). Then `duty = duty * scale / 100`.
5. **Set the PWM** with the final duty.
6. **Check SW1:** if it was not pressed last time and is pressed now (a "falling edge"), stop the motor, wait 300 ms (also hides button bounce), flip the direction, and restore the speed.
7. **Update the LCD every 5th loop** (0.5 s). Updating less often means the shared 7-segment display on the trainer flickers less.
8. **Wait 100 ms** and repeat.

### Temperature limits (the values really used in the code)

| Setting | Value in code | Meaning |
|---|---|---|
| `TEMP_START` | 260 | 26.0 deg C: at or below this, the knob gives full range |
| `TEMP_STOP` | 280 | 28.0 deg C: at or above this, the motor is stopped |

Between 26.0 and 28.0 deg C the speed shrinks in a straight line. Example: at 27.0 deg C the motor gets **half** of what the knob asks for. If the knob says 80% and the temperature is 27.0 deg C, the motor runs at 40%.

> **Heads-up:** the comments in the code say 24.0 deg C and 26.0 deg C, but the numbers `260` and `280` are actually 26.0 and 28.0 deg C. The numbers are what the program uses. Change either the numbers or the comments so they agree.

---

## 6. Things you can change (easy experiments)

| Want to... | Change this |
|---|---|
| Different temperature limits | `TEMP_START`, `TEMP_STOP` (tenths of a degree: 300 = 30.0 deg C) |
| Different PWM frequency | `PWM_LOAD` (smaller number = higher frequency) |
| Update LCD faster/slower | The `5` in `if (lcd_count >= 5)` |
| Read temperature more/less often | The `10` in `if (dht_count >= 10)` |

---

## 7. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| Build error: cannot find `tm4c123gh6pm.h` | The `inc` folder is missing from the project. Add it or fix the include path. |
| LCD is blank | Check the contrast pot, PA6/PA7, PB0-PB7 wiring and LCD power. |
| LCD shows garbage | Check the data wires PB0-PB7 are in the right order. |
| Temperature never changes (stays 0.0) | DHT11 read is failing. Check DATA wire on PE4, power, ground. It also needs 1 s after power-up. |
| Motor does not move | Check L298N power, shared GND, ENA jumper removed and wired to PF2, IN1/IN2 on PD1/PD2. |
| Motor never stops/slows with heat | Temperature may be below 26.0 deg C. Try a lower `TEMP_START` to test. |
| Knob does nothing | Check PE3 wiring: outer pins to 3.3 V and GND, middle pin to PE3. |
| LCD flickers / trainer 7-segment flickers | Normal because they share pins. LCD is updated only every 0.5 s to reduce this. |

---

## 8. Quick glossary

- **GPIO**: General Purpose Input/Output. A pin you control from code.
- **Register**: A memory box inside the chip that controls hardware.
- **ADC**: Analog to Digital Converter. Turns a voltage into a number.
- **PWM**: Pulse Width Modulation. Fast on/off to control power.
- **Duty cycle**: The percentage of time the PWM signal is "on".
- **SysTick**: A built-in timer in the ARM core, used here for delays.
- **Polling**: Repeatedly checking a pin or flag in a loop (this project uses polling, no interrupts).
