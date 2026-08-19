# Disha Smoke Monitoring System

Real-time IoT smoke detection and monitoring dashboard built for **Disha International School, Shiggaon**. The system combines an ESP8266 + MQ-2 sensor, a Google Apps Script data gateway, Google Sheets event storage, browser notifications, email alerts, and a web dashboard for live monitoring and buzzer control.

## Overview

The project is designed to monitor smoke continuously and make the sensor state visible remotely through a browser dashboard.

The system has three main layers:

```text
┌──────────────────────────────┐
│       Physical Layer         │
│                              │
│ ESP8266 + MQ-2 + Buzzer      │
└──────────────┬───────────────┘
               │ Wi-Fi / HTTPS
               ▼
┌──────────────────────────────┐
│     Google Apps Script       │
│                              │
│ • Receive sensor data        │
│ • Maintain live state        │
│ • Deduplicate events         │
│ • Store logs in Sheets       │
│ • Handle dashboard commands  │
│ • Trigger email alerts       │
└──────────────┬───────────────┘
               │ JSON / HTTPS
               ▼
┌──────────────────────────────┐
│       Web Dashboard          │
│                              │
│ • Live smoke level           │
│ • Device status              │
│ • Event history              │
│ • Buzzer controls             │
│ • Browser notifications      │
└──────────────────────────────┘
```

## Features

- **Real-time smoke monitoring** with dashboard updates every 3 seconds.
- **MQ-2 smoke/gas sensor integration** through the ESP8266 analog input.
- **Physical buzzer alarm** that can operate independently of the internet connection.
- **Remote buzzer control** from the web dashboard.
- **Automatic smoke alerts** using browser notifications and email.
- **Google Sheets logging** for historical events.
- **Server-side event deduplication** to avoid repeated log entries.
- **Heartbeat monitoring** to detect whether the IoT device is still online.
- **Wi-Fi status and RSSI reporting** on the dashboard.
- **Automatic Wi-Fi network selection** based on configured networks, signal strength, and priority.
- **Buzzer auto-reset** after smoke clears for the configured delay.
- **Recent event table** with smoke value, status, date, time, Wi-Fi, device and event reason.
- **Progressive Web App support** through `manifest.json` and a service worker.
- **Responsive web interface** suitable for monitoring from desktop or mobile browsers.

## Technology Stack

### Hardware

- ESP8266 NodeMCU
- MQ-2 smoke/gas sensor
- Buzzer
- Wi-Fi connectivity

### Firmware

- Arduino IDE
- ESP8266WiFi
- ESP8266HTTPClient
- WiFiClientSecure

### Backend / Cloud Gateway

- Google Apps Script
- Google Sheets
- Google Apps Script Properties Service
- Gmail/Google Apps Script email delivery

### Frontend

- HTML5
- CSS3
- JavaScript
- Chart.js
- Browser Notification API
- Service Worker / PWA APIs

## Repository Structure

```text
Disha Smoke Monitoring/
├── index.html          # Main monitoring dashboard
├── login.html          # Dashboard login screen
├── login.js            # Client-side login logic
├── app.js              # Dashboard logic, polling, alerts and controls
├── style.css           # Dashboard styles
├── arduino.ide         # ESP8266 firmware
├── google script       # Google Apps Script gateway and automation
├── manifest.json       # PWA manifest
├── service-worker.js   # Service worker registration/caching
├── assets/             # School/project graphics
│   ├── college.jpeg
│   ├── cursor1.png
│   └── image.png
├── fonts/              # Local typography assets
│   └── TACTICSANSEXD-BLDIT.OTF
└── README.md
```

## End-to-End Workflow

### 1. Sensor reading

The ESP8266 continuously reads the MQ-2 sensor from analog pin `A0`. The firmware averages multiple samples to reduce short-term noise.

The current firmware uses a configurable smoke threshold:

```cpp
#define MQ2_SENSOR_PIN A0
#define BUZZER_PIN D5

const int BASE_THRESHOLD = 200;
const int HYSTERESIS = 1;
```

A small hysteresis band is used to reduce rapid state changes around the threshold.

### 2. Local alarm

When smoke is detected, the ESP8266 activates the buzzer locally. This part of the alert path does **not** depend on the dashboard being online.

```text
MQ-2 → ESP8266 → Smoke detected → Buzzer ON
```

### 3. Event transmission

The ESP8266 sends data to the deployed Google Apps Script endpoint over HTTPS.

The payload contains information such as:

- Smoke value
- Smoke status
- Wi-Fi SSID
- Device ID
- Event reason
- Buzzer state
- RSSI
- Whether the event should be logged

### 4. Cloud-side processing

Google Apps Script:

- Updates the latest sensor state.
- Tracks device online/offline status.
- Deduplicates repeated events.
- Stores meaningful events in the Google Sheets `Logs` sheet.
- Stores dashboard commands.
- Sends smoke alert emails.

### 5. Dashboard polling

The browser queries the Apps Script endpoint every 3 seconds for the latest state.

The dashboard displays:

```text
Smoke Level
Intensity
System Status
Device Status
Wi-Fi
RSSI
Buzzer State
Last Event
Recent Events
```

### 6. Remote buzzer control

A user can mute or re-enable the buzzer from the dashboard.

```text
Dashboard
   │
   ├── Mute
   │      ↓
   │  Google Apps Script
   │      ↓
   │  Command state
   │      ↓
   └── ESP8266 reads command
          ↓
       Buzzer state changes
```

## Event Deduplication

One of the important reliability features is server-side event deduplication.

Google Apps Script builds an event key from the smoke value, status, and reason:

```text
smoke + status + reason
```

The same event received again within the configured short window is not written as another log entry. Heartbeats are also deliberately excluded from the event log.

This keeps the Google Sheet focused on meaningful events rather than thousands of repeated heartbeat rows.

## Device Online Detection

The dashboard does not assume that the ESP8266 is online simply because the webpage is reachable.

Instead, the Apps Script stores the timestamp of the latest update and considers the device online only when the last update is recent. The current frontend logic treats updates older than roughly **35 seconds** as offline.

```text
Recent update  → 🟢 Device Online
Old update     → 🔴 Device Offline
```

## Smoke Alert Logic

The frontend categorizes the displayed smoke reading using configured ranges. The current implementation uses:

| Smoke reading | Display state |
|---:|---|
| `<= 500` | LOW |
| `501 – 700` | MEDIUM |
| `> 700` | HIGH |

The actual smoke detection decision on the ESP8266 is based on the firmware threshold, so the display ranges and sensor threshold should be understood as separate concepts.

## Email Alerts

Google Apps Script can send an email when smoke transitions into an active state.

The alert includes information such as:

- Smoke level
- Date
- Time
- Network
- Device information

A cooldown is used to reduce repeated email notifications for the same continuing smoke event.

## Browser Notifications

The dashboard requests browser notification permission and can show an alert when a new smoke event is detected.

Notifications include:

- Smoke level
- Device/location information
- A direct indication that smoke was detected

A cooldown prevents notification spam when the same event remains active.

## Wi-Fi Management

The ESP8266 firmware can be configured with multiple known Wi-Fi networks.

At connection time it scans visible networks and chooses among configured networks using signal strength and priority.

This makes the device more resilient when multiple known networks are available.

### Important security note

The current firmware contains Wi-Fi credentials directly in the source file. **Do not publish real credentials in a production repository.** Replace them with placeholders before sharing the code publicly.

## Installation / Setup

### 1. Flash the ESP8266 firmware

Open:

```text
arduino.ide
```

in Arduino IDE.

Install/select the ESP8266 board support package and configure the correct COM port and board.

Verify the following hardware connections match the firmware:

| Component | ESP8266 pin |
|---|---|
| MQ-2 analog output | `A0` |
| Buzzer | `D5` |
| Power/GND | Appropriate supply/GND |

### 2. Configure Wi-Fi

Edit the firmware's network configuration before flashing:

```cpp
WiFiNetwork networks[] = {
  {"YOUR_WIFI_1", "YOUR_PASSWORD_1", 100},
  {"YOUR_WIFI_2", "YOUR_PASSWORD_2", 90},
};
```

### 3. Deploy Google Apps Script

Open the file:

```text
google script
```

Create a Google Apps Script project, paste the code, and deploy it as a web app.

The deployment should provide an `/exec` URL that is then used by both the firmware and frontend.

Update these values where necessary:

```text
GOOGLE_SCRIPT_URL
ALERT_EMAIL
```

The Apps Script uses a Google Sheet with a `Logs` worksheet for event history and can create supporting sheets for command logging.

### 4. Configure the dashboard

The frontend uses the Apps Script endpoint configured in `app.js`:

```javascript
const GOOGLE_SCRIPT_URL = "YOUR_DEPLOYED_APPS_SCRIPT_URL";
```

Do not publish private service URLs or credentials if the deployment is intended to be private.

### 5. Run locally

Because the frontend is static, it can be served with any static HTTP server.

For example, with Python:

```bash
python -m http.server 8000
```

Then open:

```text
http://localhost:8000/login.html
```

A local HTTP server is preferable to opening the HTML directly with `file://`, especially when using service workers and browser APIs.

## Dashboard Login

The current frontend uses a simple client-side login gate implemented in `login.js`.

The logic stores a value in `localStorage` and redirects the user to `index.html` after successful login.

### Important security limitation

This is **not real authentication**. The credentials and authentication state are implemented entirely in client-side JavaScript and local storage.

It should therefore be treated as a project/demo access screen rather than a secure authentication mechanism.

For production use, move authentication to a server-side service and protect the Apps Script/API endpoints as well.

## PWA Support

The project includes:

- `manifest.json`
- `service-worker.js`

This allows the monitoring dashboard to behave more like an installable web application on supported browsers.

## Data Model

A typical logged event contains:

```text
ID
Smoke
Status
Date
Time
WiFi
Device
Reason
ServerTime
```

The dashboard can use this history to display recent smoke events and operational status.

## Reliability Features

The firmware and cloud layer contain several mechanisms designed to keep the system usable in real-world conditions:

- Local buzzer operation even when internet connectivity is unavailable.
- Wi-Fi reconnection attempts.
- Heartbeat transmissions.
- Server-side event deduplication.
- Failed-event retry behavior in the firmware.
- Device online timeout detection.
- Buzzer auto-reset after smoke clears.
- Maintenance limits on stored event history.

## Known Limitations

This project is an educational IoT monitoring system and has several limitations that should be addressed before safety-critical deployment:

1. **Client-side login is not secure.**
2. **Wi-Fi credentials are currently embedded in firmware.**
3. **The Google Apps Script web endpoint requires careful access control for production use.**
4. **MQ-2 readings are sensor values, not calibrated professional smoke concentration measurements.**
5. **Browser notification availability depends on browser permissions and platform support.**
6. **A cloud-service outage can affect remote monitoring even though the local buzzer continues to operate.**

## Future Improvements

Potential production-oriented improvements include:

- Secure authentication with role-based access.
- Moving credentials into a secure device configuration process.
- HTTPS/API authentication and request signing.
- Multiple sensor/device management.
- SMS/WhatsApp/Telegram emergency alerts.
- Historical analytics and charts.
- Threshold configuration from the dashboard.
- Automatic device health monitoring.
- Sensor calibration and environmental compensation.
- Native mobile application support.
- Persistent cloud database instead of relying only on Google Sheets.

## Project Context

The dashboard identifies the project as a smoke monitoring system created by **Malatesh B** and **Naveen K** from **8th Alpha** at Disha International School. The interface is intended to provide a simple real-time view of smoke conditions and device status. fileciteturn38file0

## Author / Repository

GitHub: [adithybommanahalli-ui/disha_smoke-monitoring](https://github.com/adithybommanahalli-ui/disha_smoke-monitoring)

## License

No license is currently defined in the repository. If this project is intended for public reuse, add an explicit license file such as MIT before accepting external contributions.

---

**Disha Smoke Monitoring System** — an educational IoT project connecting physical smoke detection with a remotely accessible monitoring dashboard.