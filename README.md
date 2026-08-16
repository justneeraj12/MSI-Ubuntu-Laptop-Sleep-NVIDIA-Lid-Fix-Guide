# MSI Ubuntu Laptop Sleep & NVIDIA Lid Fix Guide

A comprehensive troubleshooting and permanent configuration guide for fixing sleep state issues, instant-wake bugs, and GNOME lid-switch inhibitors on MSI laptops running Ubuntu with NVIDIA hybrid graphics.

## 🔍 The Root Causes Found

1. **The NVIDIA VRAM Handshake:** Installing proprietary NVIDIA drivers breaks default sleep states unless video memory preservation (`NVreg_PreserveVideoMemoryAllocations=1`) is explicitly forced.
2. **PCIe Instant-Wake Bug:** The PCIe lanes for the NVIDIA GPU (`PEG0`, `PEG1`, `PEG2`) and the primary USB controller (`XHCI`) remain armed as system wake triggers, causing the laptop to instantly wake up milliseconds after entering sleep.
3. **GNOME Inhibitor Lock:** GNOME's power daemon (`gsd-power`) falsely flags the NVIDIA driver's display handoff as an "External monitor attached," slamming a permanent block onto the `handle-lid-switch` event.

---

## 🛠️ The Permanent Fix Installation

Follow these steps to apply a clean, isolated, and permanent fix that survives reboots and kernel updates without interfering with development workspaces or hardware peripherals.

### Step 1: Enable NVIDIA Power Services & VRAM Preservation

Ensure the NVIDIA driver properly coordinates with system suspend and saves its frame buffer:

```bash
sudo systemctl enable nvidia-suspend.service nvidia-hibernate.service nvidia-resume.service
echo "options nvidia NVreg_PreserveVideoMemoryAllocations=1" | sudo tee /etc/modprobe.d/nvidia-sleep.conf
sudo update-initramfs -u

```

### Step 2: Create a Permanent ACPI Wakeup Service

Because `/proc/acpi/wakeup` resets on every reboot, create a system service to automatically disarm faulty PCIe/USB wake triggers on startup.

1. Create the helper script:
```bash
sudo nano /etc/systemd/system-sleep-fix.sh

```


2. Paste the following script:
```bash
#!/bin/bash
# Automatically disarm rogue PCIe and USB wake triggers on boot
for dev in PEG0 PEG1 PEG2 XHCI; do
    if grep -q "$dev.*enabled" /proc/acpi/wakeup; then
        echo "$dev" > /proc/acpi/wakeup
    fi
done

```


3. Make it executable:
```bash
sudo chmod +x /etc/systemd/system-sleep-fix.sh

```


4. Create the systemd service file:
```bash
sudo nano /etc/systemd/system/msi-sleep-fix.service

```


5. Paste the service configuration:
```text
[Unit]
Description=Permanent MSI Sleep & Wakeup Fix
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/etc/systemd/system-sleep-fix.sh

[Install]
WantedBy=multi-user.target

```


6. Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable msi-sleep-fix.service

```



### Step 3: Clear the GNOME Lid Inhibitor on Login

To bypass GNOME's false external monitor lock on the lid switch without disabling your top-bar power sliders:

1. Create your user autostart directory:
```bash
mkdir -p ~/.config/autostart

```


2. Create the desktop entry script:
```bash
cat << 'EOF' > ~/.config/autostart/unblock-lid.desktop
[Desktop Entry]
Type=Application
Name=Unblock Lid Switch
Exec=pkill -f gsd-power
X-GNOME-Autostart-enabled=true
EOF

```



---

## 🚀 Verification

Reboot your system. Once you reach the desktop, verify that the rogue wakeup triggers are disarmed and the GNOME lid block is cleared:

```bash
cat /proc/acpi/wakeup | grep enabled
systemd-inhibit --list --mode=block

```

* **Expected Output:** `PEG0`, `PEG1`, `PEG2`, and `XHCI` will be absent from the enabled wakeup list, and `gsd-power` will no longer block the `handle-lid-switch`.

Close your laptop lid on battery power or plugged into a charger—it will transition smoothly into deep sleep on the first try!
