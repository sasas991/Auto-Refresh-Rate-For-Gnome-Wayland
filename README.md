# Auto-Refresh-Rate-For-Gnome-Wayland
Simple bash script that changes your refresh rate depending on ac or battery. Can improve battery life on laptops.

🔧 Automatic Refresh Rate (AC ↔ Battery)
Goal:
✔ When the laptop is plugged in → use high refresh rate (e.g., 144 Hz)
✔ When on battery → use lower refresh rate (e.g., 60 Hz)

We build this in three parts:
1. Shell script that checks AC status and sets Hz.
2. systemd service to run the script with GNOME/Wayland session context.
3. udev rule that triggers the service when power status changes.

📌 Prerequisites

✔ Linux with systemd
✔ upower installed and running (sudo dnf install upower on Fedora)
✔ A command-line tool to change refresh rate:
displayconfig-mutter (works with GNOME/Wayland; install from COPR or build),
STEP 1.
1. Create the script file
```mkdir -p ~/bin```
```nano ~/bin/power-refresh.sh```

2. Replace the refresh values and commands with your own:
#!/bin/bash

# path to AC adapter according to upower
AC_PATH="/org/freedesktop/UPower/devices/line_power_ACAD"

# check if online
ONLINE=$(upower -i "$AC_PATH" | grep "online" | awk '{print $2}')

if [[ "$ONLINE" == "yes" ]]; then
    echo "AC power — high refresh rate"
    # your high-Hz command (example for GNOME/Wayland)
    displayconfig-mutter set --connector eDP-1 --refresh-rate 144
else
    echo "On battery — low refresh rate"
    # your battery-Hz command
    displayconfig-mutter set --connector eDP-1 --refresh-rate 60
fi

3. Make it executable
chmod +x ~/bin/power-refresh.sh

STEP 2.

1. Create service file
sudo nano /etc/systemd/system/power-rate.service

2. Paste this. Adjust your "User="
[Unit]
Description=Refresh display rate on power change
After=graphical.target

[Service]
Type=oneshot
User=YOUR_USERNAME
Environment=DISPLAY=:0
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus
ExecStart=/home/YOUR_USERNAME/bin/power-refresh.sh

3. Reload systemd
sudo systemctl daemon-reload

STEP 3.

1. Create the udev rule
sudo nano /etc/udev/rules.d/99-power-switch.rules

2. Paste this
SUBSYSTEM=="power_supply", ATTR{online}=="1", TAG+="systemd", ENV{SYSTEMD_WANTS}="power-rate.service"
SUBSYSTEM=="power_supply", ATTR{online}=="0", TAG+="systemd", ENV{SYSTEMD_WANTS}="power-rate.service"

3. Reload udev
sudo udevadm control --reload-rules
sudo udevadm trigger


📌 How It Works
✔ When AC is plugged or unplugged, udev sees a change in the power_supply subsystem.
✔ udev passes the event to systemd to start power-rate.service.
✔ The service runs your script with user session context.
✔ The script checks AC status and switches refresh rate accordingly.
