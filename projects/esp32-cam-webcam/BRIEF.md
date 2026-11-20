# ESP32-CAM home webcam: brief

## What I want
A small Wi-Fi webcam I can move around the house (on a shelf, pointed out of the window at the weather), running on a battery, that I can **view from my own website (the Spoon Archive Hub)**. It's just for me to look at now and then. It has nothing to do with the hydroponics project.

## The hardware I have
- AI-Thinker style **ESP32-CAM**: the module says "ESP32-S, FCC ID 2AHMR-ESP32S, Wi-Fi + BT SoC". 8 pins on each side, one button (reset), a micro SD slot (I have a 2 GB card), an antenna connector with no antenna plugged in. The camera is most likely an OV2640 (check during setup).
- I'm not sure yet whether I have the ESP32-CAM-MB USB board. If not, programming needs a USB-serial adapter (5 V, with GPIO0 to GND while flashing).
- Battery: to be decided (e.g. an 18650 cell with a charger/boost board). I have no soldering iron yet, so start on USB power.

## My website (the Hub) and how the camera must talk to it
- The Hub runs on my Windows PC on the home network (SvelteKit, Node). I view it on the PC and, from outside, over **ZeroTier**. Nothing is port-forwarded, and the camera must **never be reachable from the internet**.
- The Hub already shows my Raspberry Pi camera by **pulling** from the device on the home network: it proxies an MJPEG stream to the page and fetches single JPEG frames on a timer. The ESP32-CAM should work the same way, so the Hub side is a small change (a separate Hub task, not part of this repo).

Build the firmware to this contract (HTTP on the home network, port 80):

| Endpoint | Returns |
|---|---|
| `GET /stream` | MJPEG, `multipart/x-mixed-replace; boundary=frame`, one JPEG per part (usable as an `<img src>`) |
| `GET /capture` | one JPEG (current frame) |
| `GET /status` | JSON: `{"name": "...", "uptime_s": 123, "rssi": -60, "battery_v": 3.9 or null, "resolution": "VGA", "mode": "live" or "snapshot", "free_heap": 12345}` |
| `GET /api/camera/settings` | `{"schema": [...], "values": {...}}`, same shape as my Pi camera API: each schema item `{"key","label","type":"choice|int|float|bool","choices"?,"min"?,"max"?,"step"?,"default"}` (resolution, quality, brightness, contrast, saturation, exposure/ae level, white balance mode, hflip, vflip) |
| `POST /api/camera/settings` | JSON partial of `values`; `200 {"values": {...}}`, or `400 {"error": "..."}`; settings saved to flash (Preferences/NVS) so they survive a reboot |

- Only one stream viewer at a time is fine (a second gets `503`).
- mDNS name like `esp32-cam.local`, so the Hub can find it without a fixed IP.
- **Battery mode ("snapshot"):** streaming all day would drain a battery in hours. Add a mode, chosen in settings, where the camera stays connected but idles at low frame rate / camera powered down between `/capture` requests, or (stretch goal) deep-sleeps and wakes every N minutes. Report `battery_v` if a battery voltage divider is wired to an ADC pin (document which pin; GPIOs are scarce on this board).
- The SD card is optional: a nice-to-have is saving one photo every N minutes to SD as a backup when the Wi-Fi is down. Never required for streaming.

## Rules
- **No secrets in git:** Wi-Fi name and password go in a git-ignored `secrets.h` (commit a `secrets.example.h` with placeholders). Never commit IP addresses, my home location, or photos from the camera.
- Keep it simple and documented for a beginner: step-by-step setup in `projects/esp32-cam-webcam/README.md` (board selection, flashing, the GPIO0 trick, what to do when it brown-outs: use a good 5 V supply).
- Antenna: on these boards the external antenna connector only works if a tiny 0 Ω resistor is moved; explain this, and use the built-in PCB antenna by default.
- Prefer Arduino framework (PlatformIO or Arduino IDE) with the official `esp32-camera` driver; start from the standard CameraWebServer approach and trim it to the contract above.
- Add a journal entry in `log/2026/` for each session (what we did, what worked, photos of the wiring are mine to add).

## Done when
1. I can open `http://esp32-cam.local/stream` on my PC and see video.
2. `/capture`, `/status` and the settings API answer as above.
3. The README lets me redo the setup from scratch.
4. Then the Hub gets its own task to add "Home cameras" (pull `/stream` and `/capture`, like the Pi camera).
