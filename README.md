# 🌱 GreenGuard — Autonomous Plant Care Controller

An Arduino UNO system that takes care of a plant's environment on its own: it
opens a **vent servo** when it gets too hot, waters via a **pump** when soil is
dry, and sounds a **buzzer alarm** when the water tank runs low — with the pump
hard-interlocked so it never runs dry. All sensors, actuators, and telemetry
run on non-blocking `millis()` scheduling.

**▶️ Try it live on Wokwi (no hardware required):** https://wokwi.com/projects/478065756491385857

## The Subsystems

| Subsystem | Sensor | Actuator | Behavior |
|---|---|---|---|
| 🌡️ Climate vent | DHT22 temperature | Servo on pin 6 | Opens (90°) at ≥ 28°C, closes at ≤ 26°C — hysteresis band |
| 💧 Irrigation | Potentiometer as soil sensor (A1) | Blue LED as pump (pin 7) | Pump ON when soil < 30%, OFF above 45% — hysteresis band |
| 🚨 Tank alarm | HC-SR04 ultrasonic on tank | Piezo buzzer (pin 8) | Beeps continuously when water < 20 cm from sensor |
| 📋 Telemetry | all of the above | Serial Monitor | CSV status row every 5 s |

## Safety: The Pump Interlock

The pump is **never allowed to run when the tank is low.** The tank check runs
first in the control logic:

```
if (tankLow)        pumpOn = false;          // hard interlock wins
else if (!pumpOn && soil < 30) pumpOn = true;
else if (pumpOn && soil > 45)  pumpOn = false;
```

Dry soil + empty tank = pump stays off, and the buzzer tells you why. This
prevents a dry-running pump (which in real hardware means burnt-out pumps).

## How It Works

The `loop()` runs four independent `millis()` timers at different rates — slow
sensors get polled less often, fast ones stay responsive:

| Task | Rate | What it does |
|---|---|---|
| DHT22 read | every 2 s | Temperature (rejected if `isnan` — keeps last good value) |
| Ultrasonic + control | every 500 ms | Tank distance, soil %, vent & pump state machines |
| Buzzer | every 400 ms toggling | Non-blocking beep while tank is low |
| CSV log | every 5 s | `ms,temp,tank_cm,soil,vent,pump,tank_low` over Serial |

### Two hysteresis loops
- **Vent:** opens ≥ 28°C, closes ≤ 26°C — a 2°C band stops the servo from
  jittering around the threshold.
- **Pump:** ON below 30% soil, OFF above 45% — forces the soil to genuinely
  re-wet before the pump stops.

### Ultrasonic water level
The HC-SR04 measures the distance from the sensor (top of tank) down to the
water surface: **larger distance = less water.** Distance > 20 cm → low tank.
A missing echo (timeout) is treated as empty, fail-safe.

## Hardware

| Part | Role |
|---|---|
| Arduino UNO | Microcontroller board |
| DHT22 | Temperature for vent control (pin 2) |
| HC-SR04 ultrasonic | Water tank level (TRIG 9, ECHO 10) |
| Potentiometer | Stands in for capacitive soil sensor (A1) |
| SG90 servo | Vent actuator (PWM pin 6) |
| Piezo buzzer | Low-water alarm (pin 8) |
| Blue LED + 220Ω | Pump indicator (pin 7) |

## Wiring

| Part | Arduino UNO Pins |
|---|---|
| DHT22 | 5V / GND / D2 |
| HC-SR04 | 5V / GND / TRIG D9 / ECHO D10 |
| Servo | 5V / GND / PWM D6 |
| Buzzer | D8 / GND |
| Pump LED (via 220Ω) | D7 → resistor → LED → GND |
| Soil pot | 5V / GND / A1 |

The exact layout is defined in [`diagram.json`](diagram.json) — open the Wokwi
link above to see the wired circuit.

## Files

- `sketch.ino` — the Arduino program (4 concurrent tasks, 2 hysteresis loops, pump interlock, CSV telemetry)
- `diagram.json` — Wokwi circuit definition (board, parts, wiring)

## Sample CSV Output

```
ms,temp,tank_cm,soil,vent,pump,tank_low
5021,24.00,8.00,52,0,0,0
10033,27.50,22.00,25,0,1,1
```

## How to Run

**Option 1 — Wokwi (easiest):** click the link above, press ▶️ Play, open the
Serial Monitor at 9600 baud. Things to try:
- Push DHT22 temp ≥ 28°C → servo swings to 90° (vent opens)
- Turn the soil pot below 30% → blue pump LED lights
- Drag HC-SR04 distance past 20 cm → buzzer beeps and the pump force-stops

**Option 2 — Real hardware:** wire the circuit as above, flash the sketch with
the Arduino IDE, open Serial Monitor at 9600 baud.

## Concepts Demonstrated

- Cooperative multi-tasking with `millis()` (4 tasks, 4 different rates)
- Hysteresis applied to *both* control loops
- Fail-safe sensor handling (`isnan`, echo timeout → treated as empty)
- Hardware interlocks (actuator blocked by safety condition)
- Ultrasonic distance sensing (`pulseIn`, µs → cm conversion)
- PWM actuator control (servo)
- Non-blocking audio (toggle-based beeping with `tone`/`noTone`)
- Structured telemetry (CSV with a header row)

## Planned Improvements

- [ ] LCD status display (like my [weather-station-datalogger](https://github.com/AkashBGBhat/weather-station-datalogger))
- [ ] Water level % instead of raw cm on the LCD
- [ ] Auto-refill / valve control for the tank
- [ ] EEPROM config for thresholds
- [ ] Real relay + pump hardware build
- [ ] ESP32 + Wi-Fi with dashboard & alerts

## Built With

- Arduino UNO (C++ / Arduino framework)
- [Wokwi](https://wokwi.com) — browser-based simulation
- Adafruit DHT + Servo libraries

## Author

**Akash Bhat** — [github.com/AkashBGBhat](https://github.com/AkashBGBhat)
