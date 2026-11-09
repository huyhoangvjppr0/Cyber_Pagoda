# Cyber Pagoda - Smart IoT Ritual & Queue Management System

> **Course:** Sensor Networks & IoT Systems  
> **Development Platform:** PlatformIO / Arduino IDE  
> **Microcontroller:** Seeed Studio XIAO ESP32-C3 Zero  
> **Dashboard UI:** Node-RED Dashboard 2.0 (`@flowfuse/node-red-dashboard`)

---

## 1. Problem Statement & Practical Applicability

In modern life, relieving psychological pressure, seeking mental balance, and experiencing gamified relaxation are increasingly popular, especially among tech-savvy youth. However, conventional interactive toy models often lack real-world system connectivity and cloud data management capabilities.

The **Cyber Pagoda** project addresses these challenges by offering:
* **Automated Queue Management:** Utilizes an ultrasonic distance sensor to regulate the flow of visitors sequentially (allowing only one person at a time on the platform, preventing congestion).
* **Digitized Traditional Rituals:** Combines physical multi-directional interactions (mechanical wooden fish striking, bell ringing, Gacha fortune-telling, and incense offerings) with real-time visual light feedback and authentic mechanical sound responses.
* **Real-time IoT Monitoring:** Synchronizes all session data, interaction statistics, and smoke/fire hazard alerts remotely via the MQTT protocol and a responsive **Node-RED Dashboard 2.0**.

---

## 2. System Architecture Block Diagram

```mermaid
flowchart TD
    subgraph INPUTS ["Sensors & Interaction Layer (Input)"]
        US["HC-SR04 Ultrasonic Sensor\n(Queue Management / Visitor Detection)"]
        MQ2["MQ-2 Smoke Sensor\n(Distinguishes Ritual Smoke vs. Fire Alarm)"]
        RTC["DS1302 RTC Module\n(Real-Time Clock for Scheduled Bell Chimes)"]
        BTN1["Button 1: Mechanical Wooden Fish"]
        BTN2["Button 2: Mechanical Bell Chime"]
        BTN3["Button 3: Gacha Fortune Draw"]
    end

    subgraph MCU ["Central Processing Unit"]
        ESP["Seeed Studio XIAO ESP32-C3 Zero\n(FSM Logic & Wi-Fi / MQTT Client)"]
    end

    subgraph OUTPUTS ["Actuators & Display Layer (Output)"]
        NEO["12-LED NeoPixel RGB Ring\n(Divine Aura & Queue Radar Scanner)"]
        LEDS["Single LEDs (with 1kΩ Resistors)\n(White: Standby | Yellow: Busy | Green: Auspicious | Red: Ominous/Fire)"]
        SERVO["SG90 Micro Servo Motor\n(Strikes Wood Fish at 15° & Bell at 165°)"]
        OLED["1.3-inch OLED Display (I2C)\n(Status, Fortunes, Session Stats)"]
    end

    subgraph IOT ["Cloud & Control Layer"]
        BROKER["MQTT Broker (Local Mosquitto / HiveMQ)"]
        DASH["Node-RED Dashboard 2.0\n(Queue Monitoring, Statistics, Alarms)"]
    end

    INPUTS --> MCU
    MCU --> OUTPUTS
    MCU <-->|MQTT Protocol (Wi-Fi)| BROKER
    BROKER <--> DASH
```

---

## 3. Bill of Materials (BOM)

All hardware components are optimized for a compact, neat, and modular assembly:

| No. | Component / Device Name | Qty | Main Functional Role |
| :---: | :--- | :---: | :--- |
| 1 | **Seeed Studio XIAO ESP32-C3 Zero** | 1 | Central microcontroller, Wi-Fi connectivity, runs PlatformIO code |
| 2 | **1.3-inch OLED Display (I2C)** | 1 | Displays status UI, fortune results, and session statistics |
| 3 | **DS1302 RTC Module** | 1 | Provides real-time clock references for automated scheduled chimes |
| 4 | **HC-SR04 Ultrasonic Sensor** | 1 | Detects the presence of a 3D visitor model (Queue management) |
| 5 | **MQ-2 Smoke Sensor (Analog)** | 1 | Measures smoke concentration (distinguishes ritual smoke vs. fire accidents) |
| 6 | **SG90 Micro Servo Motor (9g)** | 1 | Dual mechanical actuator (sweeps to 15° for wood fish, 165° for bell) |
| 7 | **8-LED NeoPixel Ring (WS2812B)** | 1 | Creates dynamic RGB divine aura lighting effects & radar scanning |
| 8 | **Single LEDs (White, Yellow, Green, Red)** | 1 each | Status indicators (each wired in series with a $1\text{k}\Omega$ current-limiting resistor) |
| 9 | **Push Buttons** | 3 | Button 1: Wood Fish; Button 2: Bell Chime; Button 3: Fortune Draw |
| 10 | **$1\text{k}\Omega$ Resistors** | 4+ | Current-limiting protection resistors for single LEDs |
| 11 | **Power Supply:** Type-C Cable & 5V Power Bank | 1 | Provides stable system power |

---

## 4. GPIO Pin Mapping on XIAO ESP32-C3 Zero

| Peripheral Device | XIAO ESP32-C3 Pin | Communication Standard | Detailed Wiring Notes |
| :--- | :---: | :---: | :--- |
| **1.3" OLED Display** | `GPIO4 (SDA)`, `GPIO5 (SCL)` | I2C | Shared I2C bus for UI rendering |
| **DS1302 RTC Module** | `GPIO4 (SDA)`, `GPIO5 (SCL)` | I2C | Shares I2C bus or mapped via separate digital pins |
| **Ultrasonic Sensor (Trig)** | `GPIO3` | Digital Out | Transmits ultrasonic pulses for visitor positioning |
| **Ultrasonic Sensor (Echo)** | `GPIO1` | Digital In | Receives reflected ultrasonic echo signals |
| **MQ-2 Smoke Sensor** | `GPIO2 (A2)` | Analog ADC | Reads analog voltage representing smoke concentration |
| **12-LED NeoPixel Ring** | `GPIO6` | One-Wire (WS2812) | Main RGB aura lighting control line |
| **SG90 Servo Motor** | `GPIO7` | PWM | Controls mechanical sweep angles (90° $\rightarrow$ 15° / 165°) |
| **White Single LED** | `GPIO10` | Digital Out | Active during Standby mode (Shrine open, via $1\text{k}\Omega$ resistor) |
| **Yellow Single LED** | `GPIO21` | Digital Out | Active when platform is busy (Queue lock, via $1\text{k}\Omega$ resistor) |
| **Green Single LED** | `GPIO0` | Digital Out | Active on Auspicious fortunes (Major/Minor Good, via $1\text{k}\Omega$ resistor) |
| **Red Single LED** | `GPIO20` | Digital Out | Active on Ominous fortunes or Smoke Fire Alarm (via $1\text{k}\Omega$ resistor) |
| **Button 1 (Wood Fish)** | `GPIO8` | Digital In (`PULLUP`) | Triggers mechanical wood fish sweep to 15° |
| **Button 2 (Bell Chime)** | `GPIO9` | Digital In (`PULLUP`) | Triggers mechanical bell chime sweep to 165° |
| **Button 3 (Fortune Draw)**| `GPIO18` | Digital In (`PULLUP`) | Triggers random Gacha fortune drawing on OLED |

---

## 5. System Workflow & MQTT Protocol

### A. Queue Finite State Machine (FSM) Lifecycle
1. **Standby State:** Ultrasonic sensor measures distance $>60\text{ cm}$. **White LED lights up**, NeoPixel ring runs blue radar animation, and Servo rests statically at **90 degrees**. OLED displays: *"Shrine Open - Welcome Next Visitor"*.
2. **Session Active State:** Places a 3D visitor model within the $20 - 45\text{ cm}$ range. **White LED turns off, Yellow LED lights up** (locking the queue). User interacts via the 3 physical push buttons:
   * **Button 1:** Servo sweeps 90° $\rightarrow$ **15°** (strikes wood fish) $\rightarrow$ returns to **90°**.
   * **Button 2:** Servo sweeps 90° $\rightarrow$ **165°** (strikes bell) $\rightarrow$ returns to **90°**.
   * **Button 3:** Generates a random fortune, updates OLED, and lights up **Green LED** (Auspicious) or **Red LED** (Ominous).
3. **Checkout State:** Removes the visitor model to $>60\text{ cm}$. **Yellow LED turns off, White LED turns back on**. Summarizes session metrics and automatically pushes a JSON payload via MQTT to Node-RED.

### B. MQTT Topics Structure (Node-RED Dashboard 2.0)
* `temple/session/stat` *(ESP32 $\rightarrow$ Node-RED)*: Transmits summarized session telemetry data when a visitor leaves the platform.
* `temple/alert/fire` *(ESP32 $\rightarrow$ Node-RED)*: Emergency alert triggered when MQ-2 smoke levels exceed safety thresholds (Flashes Red LED rapidly).
* `temple/control/bell` *(Node-RED $\rightarrow$ ESP32)*: Remote actuation command to ring the bell/wood fish via the Dashboard UI.

---

## 6. Local MQTT Broker Setup (Eclipse Mosquitto)

To ensure high stability during academic presentations without internet dependency:
1. Download and install **Eclipse Mosquitto** (Windows 64-bit).
2. Edit `mosquitto.conf` to enable external access and disable anonymous blocking:
   ```text
   listener 1883 0.0.0.0
   allow_anonymous true
   ```
3. Restart the **Mosquitto Broker** Windows Service. Get your PC's local IPv4 address via `ipconfig` in Command Prompt and configure it in your ESP32 firmware (`mqtt_server`).

---

## 7. Node-RED Dashboard 2.0 Configuration Guide

1. **Install Palette:** Go to Menu $\rightarrow$ *Manage Palette* $\rightarrow$ Install `@flowfuse/node-red-dashboard`.
2. **Create UI Pages & Groups:**
   * **Page:** `Cyber Pagoda`
   * **Group 1:** `Giám sát Hệ thống` (System Monitoring)
   * **Group 2:** `Điều khiển & Cảnh báo` (Control & Alarms)
3. **Flow Structure:**
   * **Session Statistics:** `mqtt in` (Topic: `temple/session/stat`, Output: Parsed JSON) $\rightarrow$ `ui text` (Display format: `{{msg.payload.mo_count}} gõ mõ - Quẻ: {{msg.payload.fortune}}`).
   * **Fire Alarm:** `mqtt in` (Topic: `temple/alert/fire`) $\rightarrow$ `switch` (`msg.payload.alert == true`) $\rightarrow$ `ui notification` (Top Right, Negative color).
   * **Remote Bell Control:** `ui button` (Payload JSON: `{"command": "ring_bell"}`) $\rightarrow$ `mqtt out` (Topic: `temple/control/bell`).

---

## 8. Expected Hardware Preview & Assembly Diagram
*(Take a photo of your physical breadboard circuit or Wokwi simulation schematic, upload it to your GitHub repository, and link it here using Markdown format: `![Cyber Pagoda Hardware Setup](./images/hardware_preview.png)`)*