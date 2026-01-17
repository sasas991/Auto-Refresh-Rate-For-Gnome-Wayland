```md
# Automatic Refresh Rate Switching (AC ↔ Battery) on Linux

Automatically switch your laptop display refresh rate depending on whether the system is running on **AC power** or **battery**.

Example use case:
- 🔌 **Plugged in (AC)** → 144 Hz
- 🔋 **On battery** → 60 Hz

Works reliably on **GNOME + Wayland** using `displayconfig-mutter`, and can be adapted for **Xorg** (`xrandr`).

---

## ✨ How It Works (Overview)

This setup consists of three parts:

1. **Shell script**  
   - Checks power state via `upower`
   - Applies the correct refresh rate

2. **systemd service**  
   - Runs the script inside your user’s GNOME/Wayland session

3. **udev rule**  
   - Triggers the systemd service when AC power is plugged or unplugged

This avoids common issues where udev alone cannot access the graphical session.

---

## 📋 Requirements

- Linux with **systemd**
- `upower` (usually installed by default)
- GNOME on **Wayland**
- A working refresh-rate CLI tool:
  - **GNOME/Wayland:** `displayconfig-mutter`
  - **Xorg:** `xrandr`

> ⚠️ Make sure your refresh-rate command works **manually** before automating.

---

## 🔍 Step 0 — Find Your Display Modes

List available connectors, resolutions, and refresh rates:

```bash
displayconfig-mutter list
```

Note:
- Connector name (example: `eDP-1`)
- Desired **high refresh rate** (e.g. `144`)
- Desired **low refresh rate** (e.g. `60`)

---

## 🧩 Step 1 — Create the Power Detection Script

### 1. Create a scripts directory (if it doesn’t exist)

```bash
mkdir -p ~/bin
```

### 2. Create the script

```bash
nano ~/bin/power-refresh.sh
```

### 3. Script contents

**Edit the connector name and refresh rates to match your system.**

```bash
#!/bin/bash

# Path to AC adapter (check with: upower --enumerate)
AC_PATH="/org/freedesktop/UPower/devices/line_power_ACAD"

# Check whether AC power is online
ONLINE=$(upower -i "$AC_PATH" | grep "online" | awk '{print $2}')

if [[ "$ONLINE" == "yes" ]]; then
    # Plugged in → high refresh rate
    displayconfig-mutter set --connector eDP-1 --refresh-rate 144
else
    # On battery → low refresh rate
    displayconfig-mutter set --connector eDP-1 --refresh-rate 60
fi
```

### 4. Make it executable

```bash
chmod +x ~/bin/power-refresh.sh
```

### 5. Test manually

```bash
~/bin/power-refresh.sh
```

Unplug and plug the charger to confirm it switches correctly.

---

## ⚙️ Step 2 — Create a systemd Service

This ensures the script runs **inside your GNOME session** (required for Wayland).

### 1. Create the service file

```bash
sudo nano /etc/systemd/system/power-rate.service
```

### 2. Service configuration

Replace `YOUR_USERNAME` with your actual username.

```ini
[Unit]
Description=Switch display refresh rate on power change
After=graphical.target

[Service]
Type=oneshot
User=YOUR_USERNAME
Environment=DISPLAY=:0
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus
ExecStart=/home/YOUR_USERNAME/bin/power-refresh.sh
```

> 💡 `DISPLAY` and `DBUS_SESSION_BUS_ADDRESS` are required so GNOME/Wayland accepts display changes.

### 3. Reload systemd

```bash
sudo systemctl daemon-reload
```

---

## 🔌 Step 3 — Create the udev Rule

This triggers the systemd service whenever AC power is connected or disconnected.

### 1. Create the rule file

```bash
sudo nano /etc/udev/rules.d/99-power-switch.rules
```

### 2. Rule contents

```udev
SUBSYSTEM=="power_supply", ATTR{online}=="1", TAG+="systemd", ENV{SYSTEMD_WANTS}="power-rate.service"
SUBSYSTEM=="power_supply", ATTR{online}=="0", TAG+="systemd", ENV{SYSTEMD_WANTS}="power-rate.service"
```

### 3. Reload udev

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

---

## ✅ Final Result

- 🔌 Plug in charger → **high refresh rate**
- 🔋 Unplug charger → **low refresh rate**
- No user interaction required
- Works reliably on GNOME + Wayland

---

## 🧪 Debugging

View service logs:

```bash
journalctl -u power-rate.service -f
```

List power devices (to verify AC path):

```bash
upower --enumerate
```

---

## 🖥 Xorg Alternative (Optional)

If you use **Xorg**, replace `displayconfig-mutter` with `xrandr`:

```bash
xrandr --output eDP-1 --rate 144
xrandr --output eDP-1 --rate 60
```

Everything else stays the same.

---

## 📝 Notes

- This approach is distro-agnostic (Fedora, Arch, Ubuntu, etc.)
- Wayland requires running display commands **inside the user session**
- udev alone is not sufficient for graphical actions — systemd bridges the gap

---

## 📜 License

MIT — do whatever you want 🙂
```
