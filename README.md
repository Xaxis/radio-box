# radio-box
A guide for setting up a radio box on a Raspberry Pi 5 for your station.

# Raspberry Pi 5 “Radio Box” (IC-7300) — Complete Setup Guide

A reproducible, stable, Pi 5 station computer that supports:
- **CAT / rig control** (serial CI-V over USB)
- **USB audio** (soundcard in/out to rig)
- **Digital modes** (WSJT-X, JS8Call, FLDigi)
- **Logging / utilities** (optional)
- **Headless-first** operation with optional desktop/remote GUI

This is written for high technical competence. It’s opinionated toward **stability**: predictable device names, minimal mystery services, and clean debugging paths.

---

## Table of Contents

1. Principles & Architecture  
2. Hardware & RF Hygiene  
3. IC-7300 Settings (USB CAT + USB Audio)  
4. Base OS Setup (Pi 5)  
5. Kill Serial Port Hijackers (BRLTTY)  
6. Verify USB Serial + USB Audio  
7. Permissions & Groups  
8. Stable Device Naming (udev)  
9. Rig Control Strategy (Direct vs FLRig vs rigctld)  
10. rigctld as a System Service (recommended)  
11. Audio Stack (PipeWire/Pulse) + Routing  
12. WSJT-X Setup (FT8/FT4)  
13. FLRig Setup (optional)  
14. FLDigi Suite Setup  
15. JS8Call Setup  
16. Direwolf (Packet/APRS) Setup  
17. Time Sync (Chrony, optional GPS)  
18. Remote Operation (SSH, VNC/WayVNC, NoMachine)  
19. Auto-Start + Session Hardening  
20. Troubleshooting (Serial, Audio, PTT, Splatter, “can’t key”)  
21. Backups & Migration  
22. Optional: IC-7300 over LAN (CI-V) Notes

---

## 1) Principles & Architecture

### Core stability rules
- **Never** bind configs to `/dev/ttyACM0` or “Card 2”. Those change.
- Give every critical device a **stable name**:
  - Rig serial: `/dev/icom7300`
  - Audio: explicitly selected in apps; optionally pinned by profile/routing
- Make rig control a single shared service:
  - Prefer **rigctld** (headless, scriptable, multi-app friendly)
  - Or **FLRig** if you prefer GUI-centric control

### Recommended architecture (headless-friendly)
- `rigctld` runs as a system service on TCP `localhost:4532`
- Apps connect to `Hamlib NET rigctl` → `localhost:4532`
- Audio handled by PipeWire/Pulse; route with app settings + `pavucontrol`

---

## 2) Hardware & RF Hygiene

### Required
- Raspberry Pi 5 + cooling (active recommended)
- microSD (good brand; 64GB+; 128GB fine) or USB SSD boot (optional)
- IC-7300 USB cable (USB-B on rig)
- Ethernet (recommended for setup and reliability)

### Strongly recommended RF hygiene
- Ferrite on **USB cable** near rig and near Pi
- Keep USB cable away from coax and RF hot spots
- Common-mode choking on feedlines where appropriate

---

## 3) IC-7300 Settings (USB CAT + USB Audio)

On the IC-7300, set (names may vary slightly by firmware):

- **CI-V Baud Rate**: `115200` (predictable)
- **CI-V Transceive**: `OFF` (avoid echo/duplication)
- **CI-V USB Port**: `ON`
- **DATA MOD**: `USB`
- **DATA mode for digital**: use **DATA-USB** (or ensure DATA uses USB)

PTT strategy (pick one; start with CAT):
- **PTT via CAT** (recommended): simplest with rigctld/hamlib/WSJT-X
- **RTS/DTR**: valid but adds extra serial line control config

---

## 4) Base OS Setup (Pi 5)

### 4.1 Update everything
```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

### 4.2 Install core packages (radio + tooling)
```bash
sudo apt install -y \
  git curl wget vim htop tmux \
  usbutils udev \
  pavucontrol \
  wsjtx \
  flrig fldigi flmsg flamp \
  chrony
```

Notes:
- `pavucontrol` is essential for consistent audio sanity.
- `chrony` is reliable time sync (important for FT8/FT4 decode quality).

---

## 5) Kill Serial Port Hijackers (BRLTTY)

BRLTTY often claims USB serial endpoints and breaks CAT.

```bash
sudo apt purge -y brltty
sudo reboot
```

---

## 6) Verify the rig shows up as BOTH serial + audio

Plug IC-7300 USB into Pi 5, power rig on.

### 6.1 Verify USB device presence
```bash
lsusb | grep -i icom || lsusb
```

### 6.2 Verify serial device exists
```bash
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
dmesg | tail -n 120
```

Expected commonly: `/dev/ttyACM0`

### 6.3 Verify audio device exists
```bash
aplay -l
arecord -l
```

Expected: something like “USB Audio CODEC” associated with the rig.

If serial appears but audio doesn’t (or vice versa):
- try a different cable / port
- remove hub from the chain
- re-check IC-7300 USB settings

---

## 7) Permissions & Groups

Ensure your user can access serial and audio:

```bash
sudo usermod -aG dialout,audio,plugdev "$USER"
sudo reboot
```

Confirm:
```bash
groups
```

You want `dialout` and `audio`.

---

## 8) Stable Device Naming (udev): `/dev/icom7300`

Goal: never rely on `/dev/ttyACM0` changing.

### 8.1 Find idVendor/idProduct
```bash
lsusb
```

Find the Icom line and note `idVendor` (often `0562`) and `idProduct` (`XXXX`).

### 8.2 Create udev rule
```bash
sudo nano /etc/udev/rules.d/99-icom7300.rules
```

Paste (replace `XXXX`):
```text
SUBSYSTEM=="tty", ATTRS{idVendor}=="0562", ATTRS{idProduct}=="XXXX", SYMLINK+="icom7300", GROUP="dialout", MODE="0660"
```

Reload rules:
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Verify:
```bash
ls -l /dev/icom7300
```

If it doesn’t appear, inspect attributes to match correctly:
```bash
udevadm info -a -n /dev/ttyACM0 | head -n 120
```

---

## 9) Rig Control Strategy (choose one)

### A) Direct app → rig via Hamlib
- Each app talks to `/dev/icom7300`
- Simple, but multi-app conflicts can happen

### B) FLRig as the rig “hub”
- GUI-centric; friendly; stable
- If you run a desktop session full-time, it’s excellent

### C) rigctld daemon (recommended)
- Headless-friendly; multi-app; scriptable
- Apps connect via TCP, so you can keep the rig “owned” by one daemon

This guide implements **C**. You can still install FLRig too.

---

## 10) rigctld as a System Service (recommended)

### 10.1 Find the Hamlib model for IC-7300
```bash
rigctl -l | grep -i 7300
```

You’ll see a model number (use whatever you see; call it `<MODEL>` below).

### 10.2 Sanity test direct control once
```bash
rigctl -m <MODEL> -r /dev/icom7300 -s 115200 f
```

Should return frequency (Hz).

### 10.3 Create systemd unit
```bash
sudo nano /etc/systemd/system/rigctld.service
```

Paste (replace `<MODEL>`):
```ini
[Unit]
Description=Hamlib rigctld for IC-7300
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/rigctld -m <MODEL> -r /dev/icom7300 -s 115200 -t 4532
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

Enable + start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rigctld
sudo systemctl status rigctld --no-pager
```

### 10.4 Test rigctld locally
Hamlib “NET rigctl” model is `2`:

```bash
rigctl -m 2 -r localhost:4532 f
```

If that returns the frequency, your rig control is now a stable service.

---

## 11) Audio Stack (PipeWire/Pulse) + Routing

Pi OS may use PipeWire or PulseAudio. `pavucontrol` works for both.

### 11.1 Identify sinks/sources
```bash
pactl list short sinks
pactl list short sources
```

Find the rig’s USB CODEC sink/source.

### 11.2 Use pavucontrol as your routing console
```bash
pavucontrol
```

Workflow (practical, repeatable):
1. Start your app (WSJT-X / FLDigi / JS8Call)
2. In **Playback** tab: route app output to the rig’s USB sink
3. In **Recording** tab: route app input to the rig’s USB source
4. Set levels conservatively:
   - Avoid clipping capture (RX)
   - Avoid overdriving TX audio (splatter)

### 11.3 TX audio discipline (critical)
- In WSJT-X, keep “Pwr” slider moderate
- On IC-7300, use the correct DATA/USB input and adjust so ALC barely moves (or stays minimal)
- Digital modes should be **clean**, not loud

---

## 12) WSJT-X Setup (FT8/FT4)

Open **WSJT-X → File → Settings**.

### 12.1 Radio tab (rig control)
If using **rigctld** (recommended):
- **Rig**: `Hamlib NET rigctl`
- **Network Server**: `localhost:4532`
- **PTT Method**: `CAT`
- Click **Test CAT** → should succeed
- Click **Test PTT** → should key the rig

If using direct:
- **Rig**: `Icom IC-7300`
- **Serial Port**: `/dev/icom7300`
- **Baud**: `115200`
- **Handshake**: `None` (start here)
- **PTT**: `CAT`

### 12.2 Audio tab
- Input: rig USB audio source
- Output: rig USB audio sink

### 12.3 Verify time sync (FT8 requires good time)
```bash
timedatectl status
chronyc tracking
```

If your clock is off by >1s, FT8 decode suffers.

---

## 13) FLRig Setup (optional but excellent)

If you prefer a GUI “rig console”:
- Start `flrig`
- Set:
  - Rig: `IC-7300`
  - Device: `/dev/icom7300`
  - Baud: `115200`
  - PTT: `CAT`
- Confirm frequency/mode changes reflect correctly.

You can still keep `rigctld` as the service and use FLRig only when needed, but avoid two things trying to own the same serial port at once.

---

## 14) FLDigi Suite Setup (PSK/RTTY/Olivia/etc.)

### 14.1 FLDigi rig control (choose one)
**Option A: Connect FLDigi to rigctld**
- Configure → Rig Control → Hamlib
- Use “NET rigctl” and point to `localhost:4532`

**Option B: Direct to `/dev/icom7300`**
- Configure → Rig Control → Hamlib
- Rig: IC-7300 model
- Port: `/dev/icom7300`, baud 115200

### 14.2 FLDigi audio
- Configure → Sound Card
- Select the rig USB CODEC device for capture/playback
- Use `pavucontrol` if routing gets weird

---

## 15) JS8Call Setup

### 15.1 Install (try apt first)
```bash
sudo apt install -y js8call
```

If not found on your repository, install an ARM64 build (aarch64/arm64) via `.deb` and then:
```bash
sudo apt install -y ./js8call_*.deb
```

### 15.2 JS8Call rig control
- Prefer `Hamlib NET rigctl` → `localhost:4532`
- Or direct serial `/dev/icom7300`

### 15.3 JS8Call audio
- Select rig USB CODEC for in/out
- Verify with `pavucontrol`

---

## 16) Direwolf (Packet/APRS) Setup (optional)

### 16.1 Install
```bash
sudo apt install -y direwolf
```

### 16.2 Create a starting config
Create:
```bash
nano ~/direwolf.conf
```

A minimal example (you will customize callsign, audio device, PTT strategy):
```text
ADEVICE plughw:0,0
CHANNEL 0
MYCALL YOURCALL-10
PTT RIG 0 localhost:4532
```

Notes:
- Direwolf audio device numbering varies; use `arecord -l` to identify.
- PTT can be done via rig control; rigctld-based PTT is often the cleanest.

Run:
```bash
direwolf -c ~/direwolf.conf -t 0
```

---

## 17) Time Sync (Chrony + optional GPS)

### 17.1 Chrony baseline
Chrony is installed earlier. Confirm it’s running:
```bash
systemctl status chrony --no-pager
chronyc tracking
```

### 17.2 Optional GPS time (advanced)
If you add a USB GPS:
- Use `gpsd` + chrony refclock or PPS if you care about sub-100ms accuracy.
This is optional unless you’re doing very time-sensitive work offline.

---

## 18) Remote Operation

### 18.1 SSH (mandatory baseline)
Enable SSH (if not already):
```bash
sudo systemctl enable --now ssh
```

From your Mac:
```bash
ssh <user>@<pi-hostname-or-ip>
```

### 18.2 Remote GUI options (pick one)

#### Option A: VNC (simple)
- `sudo raspi-config` → Interface Options → VNC enable
- Use a VNC client from macOS

#### Option B: NoMachine (often the nicest)
Install NoMachine (ARM64 package) and connect from macOS client.
This tends to handle audio/desktop performance very well.

#### Option C: WayVNC/Wayland (if you’re on Wayland)
Works, but you’ll already know if you want this.

---

## 19) Auto-Start + Session Hardening

### 19.1 Ensure rigctld always comes up
Already enabled via systemd. Confirm:
```bash
systemctl is-enabled rigctld
```

### 19.2 Add a simple “station health” script (optional but useful)
Create:
```bash
nano ~/station-health.sh
```

Example:
```bash
#!/usr/bin/env bash
set -euo pipefail
echo "== USB devices =="
lsusb | head -n 50
echo "== Serial =="
ls -l /dev/icom7300 || true
echo "== Audio devices =="
aplay -l || true
arecord -l || true
echo "== rigctld =="
systemctl status rigctld --no-pager || true
echo "== time =="
chronyc tracking || true
```

Enable:
```bash
chmod +x ~/station-health.sh
```

---

## 20) Troubleshooting (fast diagnosis paths)

### 20.1 “Rig won’t connect” (CAT fails)
1. Confirm device exists:
   ```bash
   ls -l /dev/icom7300
   ```
2. Confirm no other process owns it:
   ```bash
   sudo lsof /dev/icom7300 || true
   ```
3. Test direct:
   ```bash
   rigctl -m <MODEL> -r /dev/icom7300 -s 115200 f
   ```
4. If direct works but apps fail, your rig server/app settings are wrong.
5. If nothing works, re-check:
   - BRLTTY removed
   - baud rate matches rig
   - USB cable/port

### 20.2 “Audio not showing up / wrong device”
- Confirm rig appears as capture and playback:
  ```bash
  arecord -l
  aplay -l
  ```
- Route with:
  ```bash
  pavucontrol
  ```
- Verify app audio settings explicitly point to USB CODEC.

### 20.3 “PTT doesn’t key”
- If using rigctld:
  - WSJT-X radio: Hamlib NET rigctl → localhost:4532
  - PTT method: CAT
  - Test PTT
- Check rigctld is running:
  ```bash
  systemctl status rigctld --no-pager
  ```
- Confirm no second program is simultaneously owning the serial port.

### 20.4 “I transmit but I’m splattering / ALC is high”
- Reduce TX audio drive (WSJT-X “Pwr” slider, system output volume)
- On IC-7300, ensure proper DATA input and keep ALC minimal
- Digital modes should be clean and steady, not loud

### 20.5 “Device name changes / disappears after reboot”
- Fix udev rule by matching correct attributes:
  ```bash
  udevadm info -a -n /dev/ttyACM0 | less
  ```
- Prefer matching serial attributes if available (`ATTRS{serial}==...`) for bulletproof mapping.

---

## 21) Backups & Migration

Make a “radio config pack” you can move to a new Pi later.

### 21.1 Create backup directory
```bash
mkdir -p ~/radio-backup
```

### 21.2 Copy common configs (safe if missing)
```bash
cp -a ~/.local/share/WSJT-X ~/radio-backup/ 2>/dev/null || true
cp -a ~/.local/share/JS8Call ~/radio-backup/ 2>/dev/null || true
cp -a ~/.fldigi ~/.flrig ~/.flmsg ~/.flamp ~/radio-backup/ 2>/dev/null || true
cp -a ~/.asoundrc ~/radio-backup/ 2>/dev/null || true
sudo cp -a /etc/udev/rules.d/99-icom7300.rules ~/radio-backup/ 2>/dev/null || true
```

### 21.3 Tar it up
```bash
tar -czf ~/radio-backup.tgz -C ~ radio-backup
```

Copy it off the Pi:
```bash
scp <user>@<pi-ip>:~/radio-backup.tgz .
```

---

## 22) Optional: IC-7300 over LAN (CI-V) Notes

If you ever run CI-V over LAN (not USB):
- You’ll use the IC-7300 network CI-V settings and connect with hamlib to an IP/port.
- This can be great for some remote setups, but USB remains the simplest “single cable” solution for CAT + audio.

---

## Minimal “Known Good” Checklist (quick validation)

1) Rig visible:
```bash
lsusb | grep -i icom
```

2) Stable serial:
```bash
ls -l /dev/icom7300
```

3) rigctld OK:
```bash
systemctl status rigctld --no-pager
rigctl -m 2 -r localhost:4532 f
```

4) Audio OK:
```bash
arecord -l | grep -i usb
aplay -l | grep -i usb
```

5) Time OK:
```bash
chronyc tracking
```

If all 5 pass, the box is ready for real radio work.

---
