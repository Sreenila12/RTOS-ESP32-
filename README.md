# RTOS-Based Smart Home Automation with Manual Control

A smart home automation system built on ESP32 using FreeRTOS with three concurrent tasks — sensor monitoring, relay control, and button input handling. Inter-task communication is handled via FreeRTOS queues. Fully simulated and validated on Wokwi.

---

## Table of Contents
- [Components Required](#components-required)
- [Circuit Connections](#circuit-connections)
- [How the Circuit Works](#how-the-circuit-works)
- [FreeRTOS Architecture](#freertos-architecture)
- [How the Program Works](#how-the-program-works)
- [Full Code](#full-code)
- [How to Run on Wokwi](#how-to-run-on-wokwi)
- [Serial Monitor Output](#serial-monitor-output)

---

## Components Required

| Component | Quantity | Purpose |
|---|---|---|
| ESP32 DevKit V1 | 1 | Main microcontroller |
| DHT22 Sensor | 1 | Temperature and humidity monitoring |
| LED (any color) | 2 | Simulates Relay / Appliance |
| Push Button | 2 | Manual appliance ON/OFF control |
| Resistor 220Ω | 2 | Current limiting for LEDs |
| Breadboard + Wires | — | Connections |

> On Wokwi you don't need physical components — just add them from the + menu

---

## Circuit Connections

### DHT22 Sensor
```
DHT22 Pin    →    ESP32 Pin
─────────────────────────────
VCC          →    3.3V
DATA         →    GPIO 15
GND          →    GND
```

### LED 1 (Appliance 1)
```
LED Pin      →    ESP32 Pin
─────────────────────────────
+ (Anode)    →    GPIO 26  (through 220Ω resistor)
- (Cathode)  →    GND
```

### LED 2 (Appliance 2)
```
LED Pin      →    ESP32 Pin
─────────────────────────────
+ (Anode)    →    GPIO 27  (through 220Ω resistor)
- (Cathode)  →    GND
```

### Button 1 (controls LED 1)
```
Button Pin   →    ESP32 Pin
─────────────────────────────
Pin 1        →    GPIO 18
Pin 2        →    GND
```

### Button 2 (controls LED 2)
```
Button Pin   →    ESP32 Pin
─────────────────────────────
Pin 1        →    GPIO 19
Pin 2        →    GND
```

---

## How the Circuit Works

### DHT22 Sensor
- DHT22 is a digital sensor — it sends temperature and humidity data as a digital signal on the DATA pin
- ESP32 reads this signal on GPIO 15 using the DHTesp library
- VCC is connected to 3.3V (not 5V) because ESP32 is a 3.3V device

### LEDs (Relay simulation)
- In a real project, LEDs are replaced by relay modules that switch AC appliances
- On Wokwi, LEDs simulate the relay output visually
- A 220Ω resistor is placed in series with each LED to limit current and prevent burning
- When GPIO 26 goes HIGH → LED 1 turns ON (appliance ON)
- When GPIO 26 goes LOW  → LED 1 turns OFF (appliance OFF)
- Same logic applies to GPIO 27 and LED 2

### Push Buttons
- Buttons are connected with INPUT_PULLUP mode in code
- This means the pin reads HIGH by default (internal pull-up resistor active)
- When button is pressed → pin connects to GND → reads LOW
- Code detects this LOW signal as a button press event
- A 300ms debounce delay prevents multiple triggers from one press

---

## FreeRTOS Architecture

### What is FreeRTOS?
FreeRTOS is a real-time operating system for microcontrollers.
It lets you run multiple tasks "simultaneously" by switching between them
thousands of times per second — so fast it feels parallel.

### The 3 Tasks in this project

```
┌──────────────────────────────────────────────────────┐
│                   ESP32 FreeRTOS                      │
│                                                       │
│  ┌───────────────┐         ┌─────────────────────┐   │
│  │  sensorTask   │         │    buttonTask        │   │
│  │               │         │                      │   │
│  │ Reads DHT22   │         │ Reads Button 1 & 2   │   │
│  │ every 3 sec   │         │ every 50ms           │   │
│  │               │         │                      │   │
│  │ Prints temp   │         │ On press →           │   │
│  │ + humidity    │         │ sends RelayCommand   │   │
│  │ to Serial     │         │ to Queue             │   │
│  └───────────────┘         └──────────┬───────────┘   │
│                                       │               │
│                            ┌──────────▼───────────┐   │
│                            │    relayQueue        │   │
│                            │  (message box)       │   │
│                            │  holds up to 5 cmds  │   │
│                            └──────────┬───────────┘   │
│                                       │               │
│                            ┌──────────▼───────────┐   │
│                            │    relayTask         │   │
│                            │                      │   │
│                            │ Waits for message    │   │
│                            │ from queue           │   │
│                            │                      │   │
│                            │ Turns LED ON/OFF     │   │
│                            │ on GPIO 26 / 27      │   │
│                            └──────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

### Task Summary

| Task | Priority | Stack Size | What it does |
|---|---|---|---|
| sensorTask | 1 | 2048 bytes | Reads DHT22 every 3 seconds, prints to Serial |
| buttonTask | 1 | 2048 bytes | Polls buttons every 50ms, sends command to queue |
| relayTask | 2 (higher) | 2048 bytes | Waits on queue, switches LED/relay immediately |

> relayTask has priority 2 (higher than others) so it responds instantly
> when a command arrives in the queue

---

## How the Program Works

### Step 1 — setup() runs first
```
- Serial begins at 115200 baud
- DHT22 initialized on GPIO 15
- LED pins set as OUTPUT
- Button pins set as INPUT_PULLUP
- relayQueue created (holds 5 RelayCommand messages)
- All 3 tasks created and handed over to FreeRTOS scheduler
- loop() stays empty — FreeRTOS takes full control
```

### Step 2 — sensorTask runs every 3 seconds
```
- Calls dht.getTempAndHumidity()
- Prints temperature and humidity to Serial Monitor
- Calls vTaskDelay(3000ms) — gives CPU time to other tasks
- Repeat forever
```

### Step 3 — buttonTask runs every 50ms
```
- Checks if Button 1 (GPIO 18) is LOW (pressed)
    → Creates RelayCommand { relayNum=1, state=true }
    → Sends it to relayQueue using xQueueSend()
    → Waits 300ms for debounce
- Checks if Button 2 (GPIO 19) is LOW (pressed)
    → Creates RelayCommand { relayNum=2, state=true }
    → Sends it to relayQueue
    → Waits 300ms for debounce
- Calls vTaskDelay(50ms) — checks again
```

### Step 4 — relayTask waits and responds
```
- Calls xQueueReceive() — blocks here doing nothing
- When a message arrives in the queue:
    → Reads relayNum (1 or 2) and state (true/false)
    → Sets GPIO 26 HIGH/LOW (if relay 1)
    → Sets GPIO 27 HIGH/LOW (if relay 2)
    → Prints "Relay X turned ON/OFF" to Serial
- Goes back to waiting on queue
```

### Why Queue instead of a global variable?
A global variable is unsafe — if buttonTask writes at the same time
relayTask reads, data gets corrupted (race condition).
A FreeRTOS queue is thread-safe by design — one task writes,
one task reads, no corruption possible.

---

## Full Code

```cpp
#include <Arduino.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include <DHTesp.h>

// ─── Pin Definitions ───────────────────────────────────
#define DHT_PIN      15
#define RELAY1_PIN   26
#define RELAY2_PIN   27
#define BUTTON1_PIN  18
#define BUTTON2_PIN  19

DHTesp dht;
QueueHandle_t relayQueue;

// ─── Command structure sent through queue ──────────────
typedef struct {
  int  relayNum;   // 1 or 2
  bool state;      // true = ON, false = OFF
} RelayCommand;

// ─── TASK 1: Read sensor every 3 seconds ───────────────
void sensorTask(void *param) {
  while (true) {
    TempAndHumidity data = dht.getTempAndHumidity();
    Serial.printf("Temp: %.1f°C  Humidity: %.1f%%\n",
                   data.temperature, data.humidity);
    vTaskDelay(pdMS_TO_TICKS(3000));
  }
}

// ─── TASK 2: Control relay based on queue message ──────
void relayTask(void *param) {
  RelayCommand cmd;
  while (true) {
    // Blocks here until a message arrives — uses 0 CPU while waiting
    if (xQueueReceive(relayQueue, &cmd, portMAX_DELAY)) {
      int pin = (cmd.relayNum == 1) ? RELAY1_PIN : RELAY2_PIN;
      digitalWrite(pin, cmd.state ? HIGH : LOW);
      Serial.printf("Relay %d turned %s\n",
                     cmd.relayNum, cmd.state ? "ON" : "OFF");
    }
  }
}

// ─── TASK 3: Poll buttons and send commands to queue ───
void buttonTask(void *param) {
  RelayCommand cmd;
  while (true) {
    if (digitalRead(BUTTON1_PIN) == LOW) {
      cmd = {1, true};
      xQueueSend(relayQueue, &cmd, 0);
      vTaskDelay(pdMS_TO_TICKS(300));  // debounce
    }
    if (digitalRead(BUTTON2_PIN) == LOW) {
      cmd = {2, true};
      xQueueSend(relayQueue, &cmd, 0);
      vTaskDelay(pdMS_TO_TICKS(300));  // debounce
    }
    vTaskDelay(pdMS_TO_TICKS(50));  // poll every 50ms
  }
}

// ─── SETUP ─────────────────────────────────────────────
void setup() {
  Serial.begin(115200);
  dht.setup(DHT_PIN, DHTesp::DHT22);

  pinMode(RELAY1_PIN,  OUTPUT);
  pinMode(RELAY2_PIN,  OUTPUT);
  pinMode(BUTTON1_PIN, INPUT_PULLUP);
  pinMode(BUTTON2_PIN, INPUT_PULLUP);

  // Create queue — holds up to 5 RelayCommand messages
  relayQueue = xQueueCreate(5, sizeof(RelayCommand));

  // Create all 3 tasks
  xTaskCreate(sensorTask, "Sensor", 2048, NULL, 1, NULL);
  xTaskCreate(relayTask,  "Relay",  2048, NULL, 2, NULL);
  xTaskCreate(buttonTask, "Button", 2048, NULL, 1, NULL);

  // loop() intentionally left empty — FreeRTOS scheduler takes over
}

void loop() {
  // Empty — FreeRTOS manages everything
}
```

---

## How to Run on Wokwi

1. Go to [wokwi.com](https://wokwi.com)
2. Click **New Project** → select **ESP32**
3. Click **+** and add:
   - DHT22
   - 2× LED
   - 2× Push Button
   - 2× Resistor (set value to 220 in properties)
4. Connect all wires as per the circuit table above
5. Click on `sketch.ino` → paste the full code above
6. Click on `libraries.txt` → add: `DHTesp`
7. Click ▶ **Play**
8. Open **Serial Monitor** at bottom
9. Press Button 1 → LED 1 turns ON, Serial shows "Relay 1 turned ON"
10. Press Button 2 → LED 2 turns ON, Serial shows "Relay 2 turned ON"
11. Temperature and humidity print every 3 seconds automatically

---

## Serial Monitor Output

```
Temp: 28.4°C  Humidity: 65.2%
Relay 1 turned ON
Temp: 28.5°C  Humidity: 65.1%
Relay 2 turned ON
Temp: 28.4°C  Humidity: 65.3%
Relay 1 turned OFF
```

---

## Key FreeRTOS Concepts Used

| Concept | Function Used | Purpose |
|---|---|---|
| Task creation | xTaskCreate() | Spawns independent concurrent tasks |
| Queue creation | xQueueCreate() | Creates thread-safe message channel |
| Send to queue | xQueueSend() | buttonTask sends relay command |
| Receive from queue | xQueueReceive() | relayTask waits and receives command |
| Non-blocking delay | vTaskDelay() | Gives CPU to other tasks during wait |
| Tick conversion | pdMS_TO_TICKS() | Converts milliseconds to RTOS ticks |

---

## Future Improvements

- Add MQTT integration for remote control via HiveMQ broker
- Integrate Blynk mobile dashboard for appliance scheduling
- Add Google Home / Alexa voice control via IFTTT
- Replace LEDs with actual relay modules for real appliances
- Add toggle logic (press once = ON, press again = OFF)

---

## Author

**Sreenila K A**
B.Tech Electronics and Communication Engineering
Federal Institute of Science and Technology, Angamaly
GitHub: [github.com/Sreenila12](https://github.com/Sreenila12)
