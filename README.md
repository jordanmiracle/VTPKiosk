# VTP-1000 Kiosk

A custom Android kiosk application built for the **Atlona AT-VTP-1000VL** touchpanel, repurposed from a commercial AV control panel into a full-screen Home Assistant dashboard.

---

## Hardware

| Property | Value |
|---|---|
| Device | Atlona AT-VTP-1000VL |
| OEM Manufacturer | Prodvx (Shenzhen) |
| SoC | Rockchip RK3288 (board: rk30sdk) |
| Architecture | armeabi-v7a (32-bit ARM) |
| Android Version | 8.1.0 (Oreo) |
| API Level | 27 |
| Display | 10" 1280x800 capacitive touchscreen |
| Connectivity | PoE (802.3at), USB, Ethernet |
| Panel IP | 192.168.1.113 (set static in your router) |

### Notable System Apps
- `com.velocity` — Original Atlona Velocity control app (disabled)
- `com.example.leddemo` — Controls the LED ring around the bezel (located at `/oem/bundled_persist-app/LedDemo/`)
- `android.rockchip.update.service` — Rockchip OTA update service
- `com.z.settingnov7` — Prodvx scheduling app (system)

---

## What This App Does

- Launches fullscreen in immersive mode (no nav bar, no status bar)
- Loads a configured URL (Home Assistant dashboard) in a locked WebView
- Keeps the screen on permanently
- Registers itself as the default launcher so it survives reboots
- Back button navigates within the WebView rather than exiting the app

---

## Prerequisites

- [Android Studio](https://developer.android.com/studio) (latest stable)
- ADB (Android Debug Bridge) — installed via Android Studio SDK Manager
  - SDK Tools → Android SDK Platform-Tools
- The panel must have USB Debugging enabled (Developer Options)
- Home Assistant instance running and accessible on your LAN

---

## ADB Setup

### Add platform-tools to PATH (Windows)

Add the following to your user `PATH` environment variable:
```
%LOCALAPPDATA%\Android\Sdk\platform-tools
```

Or navigate directly:
```powershell
cd C:\Users\<username>\AppData\Local\Android\Sdk\platform-tools
```

### Connect to the Panel

The panel connects most reliably via **ADB over TCP/IP** (network), not USB.

**First time setup (requires USB):**
```powershell
.\adb.exe devices                  # confirm USB connection
.\adb.exe tcpip 5555               # enable TCP mode
.\adb.exe connect 192.168.1.113    # connect over network
```

**Subsequent connections (network only):**
```powershell
.\adb.exe connect 192.168.1.113
```

**If connection is refused** (TCP mode resets on reboot):
```powershell
.\adb.exe kill-server
.\adb.exe start-server
.\adb.exe connect 192.168.1.113
```

**Enable Developer Options on the panel:**
1. Swipe from left edge → Advanced Setup → PIN: `554361`
2. Navigate to About → tap Build Number 7 times
3. Developer Options → Enable USB Debugging

---

## Project Setup

### Clone and Open

```bash
git clone https://github.com/<your-username>/vtp-kiosk.git
```

Open in Android Studio: **File → Open** → select the project folder.

Let Gradle sync complete before building.

### Configure the Target URL

In `app/src/main/java/com/example/vtpkiosk/MainActivity.kt`, update:

```kotlin
webView.loadUrl("http://192.168.1.124:8123")
```

Replace `192.168.1.124` with the LAN IP of your Home Assistant instance.

---

## Building and Deploying

### Via Android Studio (recommended)

1. Connect the panel via ADB (see above)
2. Confirm it appears in **Device Manager** (API 27, Physical, arm)
3. Hit **▶ Run** and select the Atlona panel as the target device

### Via ADB manually

Build the APK via **Build → Build Bundle(s)/APK(s) → Build APK(s)**, then:

```powershell
.\adb.exe install app\build\outputs\apk\debug\app-debug.apk
```

---

## Home Assistant (Docker)

HA runs in a Docker container on a local machine for development/testing.

### Run the Container (Windows)

```powershell
docker run -d `
  --name homeassistant `
  --restart=unless-stopped `
  -e TZ=America/Los_Angeles `
  -v C:\homeassistant:/config `
  -p 8123:8123 `
  ghcr.io/home-assistant/home-assistant:stable
```

> **Note:** Use `-p 8123:8123` on Windows — `--network=host` does not work with Docker Desktop on Windows.

### Allow Through Windows Firewall

Run in PowerShell as Administrator:
```powershell
New-NetFirewallRule -DisplayName "Home Assistant" -Direction Inbound -Protocol TCP -LocalPort 8123 -Action Allow
```

### Access

- From the Docker machine: `http://localhost:8123`
- From the panel or other LAN devices: `http://192.168.1.124:8123`

---

## Key Files

```
app/
├── src/main/
│   ├── AndroidManifest.xml       # Permissions, launcher registration
│   ├── java/com/example/vtpkiosk/
│   │   └── MainActivity.kt       # WebView kiosk logic
│   └── res/                      # Resources (mostly unused)
└── build.gradle                  # Min SDK 27, target SDK 27
```

---

## Known Issues & Notes

- **Velocity splash screen** persists briefly on boot — this is a system-level Rockchip/Prodvx boot animation baked into the firmware, not the Velocity app itself. Cosmetic only.
- **TCP ADB resets on reboot** — you must re-run `adb tcpip 5555` via USB after a panel reboot, or set it in a boot script.
- **`--network=host` broken on Windows Docker Desktop** — always use explicit port mapping (`-p 8123:8123`) on Windows.
- **KioskZen incompatible** — crashes on Android 8.1 (API 27) due to newer API dependencies.
- **No Play Store** — all apps must be sideloaded via ADB. The system includes `com.android.apkinstaller` for local installs if needed.
- **Treble enabled** — GSI flashing is possible but arm32 limits viable options to Android 9/10 PHH builds. Not recommended unless you have a specific reason.

---

## Future Ideas

- [ ] Integrate LED ring (`com.example.leddemo`) with Home Assistant as a status indicator
- [ ] Add Spotify / local media player card in HA dashboard  
- [ ] Import Google Home devices via HA Google integration
- [ ] Migrate HA to dedicated Raspberry Pi 4 for always-on reliability
- [ ] Explore AWS/Nabu Casa for remote access
- [ ] Investigate ADB TCP persistence across reboots via Rockchip init scripts
- [ ] Reverse engineer LedDemo APK for direct LED control from HA automations

---

## References

- [Atlona VTP-1000VL Product Page](https://atlona.com/product/vtp-1000vl/)
- [Home Assistant Docker Install](https://www.home-assistant.io/installation/linux#docker-compose)
- [PHH Treble GSI (arm32)](https://github.com/phhusson/treble_experimentations)
- [ADB Documentation](https://developer.android.com/tools/adb)
