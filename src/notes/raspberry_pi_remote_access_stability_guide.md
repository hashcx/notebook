# Raspberry PI Remote Access Stability Guide

A complete, step-by-step reference guide to preventing connection drops, handling hard freezes, configuring reboot email alerts, auto-reconnecting Wi-Fi, and disabling power saving on Raspberry Pi OS.

---

## Table of Contents
1. [Hardware Watchdog Timer Setup](#hardware-watchdog-timer-setup)
2. [Outbound Email Configuration (`msmtp`)](#outbound-email-configuration-msmtp)
3. [Automated Reboot Email Notifications](#automated-reboot-email-notifications)
4. [Automatic Wi-Fi Reconnect Script](#automatic-wi-fi-reconnect-script)
5. [Disabling Wi-Fi Power Saving](#disabling-wi-fi-power-saving)
6. [Troubleshooting Quick Reference](#troubleshooting-quick-reference)

---

## 1. Hardware Watchdog Timer Setup

The Broadcom SoC includes a built-in hardware watchdog timer. If Raspberry Pi OS hangs or freezes completely, the hardware timer forcibly resets the system.

### Step 1: Configure `systemd` Watchdog Integration
Edit the primary `systemd` configuration file:
```bash
sudo nano /etc/systemd/system.conf
```

Uncomment and configure the runtime watchdog settings:
```ini
RuntimeWatchdogSec=15s
RebootWatchdogSec=5min
```
* **`RuntimeWatchdogSec=15s`**: Systemd pings `/dev/watchdog` every ~7.5s. If the OS freezes for 15s, the hardware reboots the board.

### Step 2: Enable Hardware Module in Firmware Config
Ensure the hardware watchdog overlay is active in `/boot/firmware/config.txt` (or `/boot/config.txt` on older OS releases):
```bash
sudo nano /boot/firmware/config.txt
```
Add the following line if missing:
```text
dtparam=watchdog=on
```

### Step 3: Reboot and Verify
Reboot the Raspberry Pi:
```bash
sudo reboot
```

After rebooting, verify that the device node exists:
```bash
ls -l /dev/watchdog*
```
*Expected Output:* `/dev/watchdog` and `/dev/watchdog0`

### Step 4: Test Watchdog (Optional)
> **Warning:** This forces an immediate kernel panic to trigger a hard reset.
```bash
sudo bash -c 'echo c > /proc/sysrq-trigger'
```

---

## 2. Outbound Email Configuration (`msmtp`)

`msmtp` is a lightweight replacement for `sendmail`/`postfix` that forwards outgoing emails via an external SMTP server (e.g., Gmail).

### Step 1: Install Required Packages
```bash
sudo apt update
sudo apt install -y msmtp msmtp-mta bsd-mailx
```

### Step 2: Configure System Configuration `/etc/msmtprc`
Create/edit `/etc/msmtprc`:
```bash
sudo nano /etc/msmtprc
```

Add the following configuration:
```text
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /var/log/msmtp.log

account        default
host           smtp.gmail.com
port           587
from           your_email@gmail.com
user           your_email@gmail.com
password       your_app_password
```
*> **Note for Gmail users:** Generate an **App Password** under Google Account Security settings (requires 2-Step Verification).*

### Step 3: Configure File and Log Permissions
Ensure all local users and root can read configuration and write to the log file:
```bash
# Allow read permissions on system configuration
sudo chmod 644 /etc/msmtprc

# Setup log file permissions
sudo touch /var/log/msmtp.log
sudo chmod 666 /var/log/msmtp.log
```

### Step 4: Test Sending Email
```bash
echo "Test message body" | mail -s "Test Subject" destination@example.com
```

---

## 3. Automated Reboot Email Notifications

Receive an automated email alert with system metrics (IP address, hostname, uptime, CPU temp, RAM, and disk usage) whenever the Pi boots up.

### Step 1: Create the Alert Script
```bash
sudo nano /usr/local/bin/send_boot_email.sh
```

Paste the script content:
```bash
#!/bin/bash

# Pause 15 seconds to ensure network and DNS are active
sleep 15

RECIPIENT="your_email@gmail.com"
HOSTNAME=$(hostname)
IP_ADDRESS=$(hostname -I | awk '{print $1}')
DATE_TIME=$(date)
UPTIME=$(uptime -p)

# Gather system metrics
CPU_TEMP=$(vcgencmd measure_temp 2>/dev/null | cut -d'=' -f2 || echo "N/A")
RAM_USAGE=$(free -h | awk '/Mem:/ {print $3 "/" $2}')
DISK_USAGE=$(df -h / | awk 'NR==2 {print $3 "/" $2 " (" $5 ")"}')

SUBJECT="🚨 [Alert] $HOSTNAME Booted Up"

BODY=$(cat <<EOF
Your Raspberry Pi has successfully completed boot setup.

System Diagnostics:
----------------------------------
Hostname:     $HOSTNAME
Local IP:     $IP_ADDRESS
Boot Time:    $DATE_TIME
Uptime:       $UPTIME
CPU Temp:     $CPU_TEMP
RAM Usage:    $RAM_USAGE
Disk Space:   $DISK_USAGE
----------------------------------
EOF
)

echo "$BODY" | mail -s "$SUBJECT" "$RECIPIENT"
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/send_boot_email.sh
```

### Step 2: Create systemd Startup Service
```bash
sudo nano /etc/systemd/system/boot-email.service
```

Paste the following service configuration:
```ini
[Unit]
Description=Send email notification on boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/send_boot_email.sh
User=root

[Install]
WantedBy=multi-user.target
```

Enable and activate:
```bash
sudo systemctl daemon-reload
sudo systemctl enable boot-email.service
```

---

## 4. Automatic Wi-Fi Reconnect Script

Automatically checks connectivity every 5 minutes and restarts NetworkManager or interface if connection drops.

### Step 1: Create the Reconnect Script
```bash
sudo nano /usr/local/bin/wifi_reconnect.sh
```

Paste the following:
```bash
#!/bin/bash

TARGET_IP="8.8.8.8"
INTERFACE="wlan0"
LOG_FILE="/var/log/wifi_reconnect.log"

# Ping target 3 times; if no response, initiate reconnection
if ! ping -c 3 -W 5 "$TARGET_IP" > /dev/null 2>&1; then
    echo "$(date): Wi-Fi connection lost. Attempting reconnect on $INTERFACE..." >> "$LOG_FILE"
    
    if command -v nmcli > /dev/null 2>&1; then
        nmcli device disconnect "$INTERFACE"
        sleep 2
        nmcli device connect "$INTERFACE"
    else
        ifconfig "$INTERFACE" down
        sleep 5
        ifconfig "$INTERFACE" up
    fi

    sleep 10
    if ping -c 3 -W 5 "$TARGET_IP" > /dev/null 2>&1; then
        echo "$(date): Wi-Fi successfully reconnected." >> "$LOG_FILE"
    else
        echo "$(date): Reconnection failed. Restarting NetworkManager..." >> "$LOG_FILE"
        systemctl restart NetworkManager 2>/dev/null || systemctl restart dhcpcd 2>/dev/null
    fi
fi
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/wifi_reconnect.sh
```

### Step 2: Schedule Cron Job
Open root crontab:
```bash
sudo crontab -e
```

Add to the bottom:
```text
*/5 * * * * /usr/local/bin/wifi_reconnect.sh
```

---

## 5. Disabling Wi-Fi Power Saving

Prevents Wi-Fi adapter from entering power-saving sleep state.

### For NetworkManager (Raspberry Pi OS Bookworm / Bullseye - Recommended)
1. Create NetworkManager override file:
   ```bash
   sudo nano /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf
   ```
2. Add the following config (`2` disables power save):
   ```ini
   [connection]
   wifi.powersave = 2
   ```
3. Restart NetworkManager:
   ```bash
   sudo systemctl restart NetworkManager
   ```
4. Check power save status:
   ```bash
   /sbin/iw dev wlan0 get power_save
   ```
   *Output should show:* `Power save: off`

---

## 6. Troubleshooting Quick Reference

| Issue | Cause | Fix |
|---|---|---|
| **msmtp error code 78** | Config syntax error or duplicate account declaration | Check `/etc/msmtprc` for syntax errors or `account default` duplicated lines. |
| **msmtp Permission denied on `/etc/msmtprc`** | File permissions set to `600` owned by root | Run `sudo chmod 644 /etc/msmtprc`. |
| **msmtp cannot open `/var/log/msmtp.log`** | Non-root users cannot write to log file | Run `sudo chmod 666 /var/log/msmtp.log`. |
| **systemd 203/EXEC error for `iw`** | `/usr/bin/iw` path does not exist on Pi OS | Use NetworkManager config `/etc/NetworkManager/conf.d/default-wifi-powersave-on.conf` instead, or binary at `/sbin/iw`. |
