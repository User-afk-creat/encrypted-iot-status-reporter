# 🔐 Encrypted IoT Status Reporter

A free Wokwi simulation project combining **WiFi networking**, **HMAC-SHA256 authenticated security**, and a full **Blynk.io cloud dashboard** with notifications, automation, and a live terminal log — all on a virtual ESP32.

---

## 📋 Project Overview

The ESP32 connects to Wokwi's virtual WiFi, computes an **HMAC-SHA256** digest of a timestamped status message, and pushes data to Blynk.io in real time. The dashboard provides live visibility into message integrity, device uptime, and security alerts.

---

## 🎯 Features

- ✅ WiFi connectivity via `Wokwi-GUEST` (no password required)
- ✅ **HMAC-SHA256** authentication using the built-in `mbedtls` library
- ✅ **Blynk Notification** — alerts when the HMAC digest changes unexpectedly
- ✅ **Blynk Automation** — triggers actions when uptime crosses a threshold
- ✅ **Blynk Terminal widget** — live scrolling log of digests and events
- ✅ No external hardware needed — ESP32 only

---

## 🛠️ Hardware Setup (`diagram.json`)

| Component | Details |
|-----------|---------|
| **Board** | ESP32 DevKit V1 |
| **WiFi** | Internal `Wokwi-GUEST` network (no password) |
| **Cloud Platform** | Blynk.io (free tier) |

---

## ☁️ Blynk.io Setup

### 1. Create Template & Device
1. Sign up at [blynk.io](https://blynk.io)
2. Create a new **Template** → add a **Device**
3. Copy the **Auth Token** and **Template ID**

### 2. Configure Datastreams

| Virtual Pin | Name | Data Type | Purpose |
|-------------|------|-----------|---------|
| `V0` | Status Message | String | Raw timestamped payload |
| `V1` | HMAC-SHA256 | String | 64-char hex digest |
| `V2` | Uptime (s) | Integer | Device uptime in seconds |
| `V3` | Terminal | String | Live log feed |
| `V4` | Alert | Integer | Notification trigger (reserved) |

### 3. Add Dashboard Widgets

| Widget | Pin | Purpose |
|--------|-----|---------|
| Label | V0 | Display current status message |
| Label | V1 | Display current HMAC digest |
| Gauge / Value | V2 | Show device uptime |
| Terminal | V3 | Live scrolling digest log |

### 4. Set Up Blynk Notification
1. In the Blynk Console, go to **Events** and create a new event with the name `hmac_change`
2. Set the display name to something like "Security Alert: HMAC Changed"
3. Enable **Push Notifications** for this event
4. The firmware calls `Blynk.logEvent("hmac_change", ...)` automatically when a digest mismatch is detected

### 5. Set Up Blynk Automation
1. Go to **Automations** in the Blynk Console
2. Create a new automation:
   - **Trigger**: Datastream `V2` (Uptime) ≥ `3600` (1 hour)
   - **Action**: Send notification — "Device has been running for 1 hour"
3. No additional firmware code is needed — Blynk handles this server-side

---

## 💻 Code (`sketch.ino`)

> Replace `YOUR_TEMPLATE_ID` and `YOUR_BLYNK_AUTH_TOKEN` before running.

```cpp
#define BLYNK_TEMPLATE_ID   "YOUR_TEMPLATE_ID"
#define BLYNK_TEMPLATE_NAME "IoT Status Reporter"
#define BLYNK_AUTH_TOKEN    "YOUR_BLYNK_AUTH_TOKEN"

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include "mbedtls/md.h"

const char* ssid     = "Wokwi-GUEST";
const char* password = "";
char auth[]          = BLYNK_AUTH_TOKEN;

// HMAC Secret Key (keep private in real projects!)
const char* HMAC_KEY = "MySecretIoTKey123";

String lastHash  = "";
bool   notifSent = false;
WidgetTerminal terminal(V3);

String computeHMAC(const String& message) {
  byte result[32];
  mbedtls_md_context_t ctx;
  const mbedtls_md_info_t* info = mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);

  mbedtls_md_init(&ctx);
  mbedtls_md_setup(&ctx, info, 1); // 1 = HMAC mode
  mbedtls_md_hmac_starts(&ctx, (const unsigned char*)HMAC_KEY, strlen(HMAC_KEY));
  mbedtls_md_hmac_update(&ctx, (const unsigned char*)message.c_str(), message.length());
  mbedtls_md_hmac_finish(&ctx, result);
  mbedtls_md_free(&ctx);

  String hex = "";
  for (int i = 0; i < 32; i++) {
    char buf[3];
    sprintf(buf, "%02x", result[i]);
    hex += buf;
  }
  return hex;
}

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
  Serial.println("\nConnected! IP: " + WiFi.localIP().toString());

  Blynk.config(auth);
  Blynk.connect();

  terminal.clear();
  terminal.println("=== IoT Reporter Started ===");
  terminal.flush();
}

void loop() {
  Blynk.run();

  unsigned long uptimeSec = millis() / 1000;
  String data    = "System_Status_OK_Time_" + String(millis());
  String hmacHex = computeHMAC(data);

  // Notification: alert on unexpected HMAC change
  if (lastHash != "" && hmacHex != lastHash && !notifSent) {
    Blynk.logEvent("hmac_change", "ALERT: HMAC changed unexpectedly!");
    terminal.println("[WARN] HMAC mismatch detected!");
    terminal.flush();
    notifSent = true;
  } else {
    notifSent = false;
  }
  lastHash = hmacHex;

  // Publish to Blynk
  Blynk.virtualWrite(V0, data);
  Blynk.virtualWrite(V1, hmacHex);
  Blynk.virtualWrite(V2, uptimeSec);   // Automation watches this

  // Terminal log
  terminal.print("[" + String(uptimeSec) + "s] ");
  terminal.println(hmacHex.substring(0, 16) + "...");
  terminal.flush();

  delay(5000);
}
```

---

## 💡 Key Concepts

### 🌐 WiFi Networking
Wokwi simulates a full WiFi stack. The virtual ESP32 receives a real IP address and connects to external services including the Blynk cloud, making the simulation behaviorally identical to a physical deployment.

### 🔒 HMAC-SHA256
Unlike plain SHA-256, **HMAC** (Hash-based Message Authentication Code) combines the message with a **secret key** before hashing. This means that even if an attacker knows the algorithm, they cannot forge a valid hash without knowing the key — providing both **integrity** and **authenticity** guarantees.

| Method | Integrity | Authenticity |
|--------|-----------|--------------|
| SHA-256 (plain) | ✅ | ❌ |
| HMAC-SHA256 | ✅ | ✅ |

### 🔔 Blynk Notification
When the firmware detects that the HMAC digest has changed between cycles (which in a real system would indicate tampering), it calls `Blynk.logEvent()` to fire a push notification to your phone via the Blynk app.

### ⚙️ Blynk Automation
The device uptime (V2) is published every 5 seconds. A server-side Blynk Automation rule monitors this value and fires a notification when it crosses a configured threshold — for example, alerting you when the device has been online for 1 hour. No extra firmware code is required.

### 📟 Blynk Terminal
The Terminal widget on V3 acts as a live serial-style log inside the Blynk dashboard, displaying a timestamped stream of HMAC digest previews and any warning events.

---

## ⚠️ Free-Tier Limitations

| Platform | Limitation |
|----------|------------|
| **Wokwi (Free)** | No private gateway — cannot browse to ESP32 IP from local machine |
| **Wokwi (Free)** | All projects are public — never embed real keys or tokens |
| **Blynk (Free)** | Maximum 2 devices and limited datastreams |

---

## 🚀 Getting Started

1. Sign up at [blynk.io](https://blynk.io) and create a Template + Device
2. Create the 5 datastreams (V0–V4) and add dashboard widgets
3. Create the `hmac_change` event and Automation rule in the Blynk Console
4. Open [Wokwi.com](https://wokwi.com) and create a new ESP32 project
5. Paste `sketch.ino` and fill in your Auth Token, Template ID, and HMAC key
6. Add the `BlynkSimpleEsp32` library via Wokwi's library manager
7. Click ▶ **Start Simulation** — watch live data on your Blynk dashboard

---

## 📚 References

- [Blynk.io Documentation](https://docs.blynk.io)
- [Blynk Events & Notifications](https://docs.blynk.io/en/blynk.console/events)
- [Blynk Automations](https://docs.blynk.io/en/concepts/automations)
- [Wokwi ESP32 WiFi Guide](https://docs.wokwi.com/guides/esp32-wifi)
- [mbedTLS HMAC API](https://mbed-tls.readthedocs.io/en/latest/)

---
## Simulate an alert
```
void loop() {
  Blynk.run();

  unsigned long uptimeSec = millis() / 1000;
  
  // 1. Create original data and its AUTHENTIC hash
  String originalData = "System_Status_OK_Time_" + String(millis());
  String authorizedHash = computeHMAC(originalData);

  // 2. SIMULATE ATTACK: Change the message, but keep the OLD hash
  String tamperedData = "System_Status_CRITICAL_Time_" + String(millis()); 

  // 3. THE VERIFICATION (This is what your NIDS would do)
  // Check if the hash of what we are sending matches the hash we calculated
  String currentActualHash = computeHMAC(tamperedData);

  if (currentActualHash != authorizedHash) {
    // This is a TRUE detection: The data and the hash don't match!
    Blynk.logEvent("hmac_change", "ALERT: Data Integrity Compromised!");
    Blynk.virtualWrite(V4, 1);
    terminal.println("[CRITICAL] HMAC Mismatch! Data was altered.");
  } else {
    Blynk.virtualWrite(V4, 0);
  }

  // Send the "Tampered" data and the "Old" hash to Blynk to show the failure
  Blynk.virtualWrite(V0, tamperedData);
  Blynk.virtualWrite(V1, authorizedHash);
  Blynk.virtualWrite(V2, uptimeSec);

  terminal.flush();
  delay(5000);
}

