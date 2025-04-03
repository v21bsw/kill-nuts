# 🖥️ Power-Check Scripts for Proxmox and VMs  

### **Automated Shutdown for Proxmox and VMs in Case of Power Failure**  

This repository contains scripts that automate the **graceful shutdown** of Proxmox hosts and virtual machines when a **UPS (Uninterruptible Power Supply) goes on battery** and the **UPS-hosting VM shuts down**.  

## 📌 Features  

✅ **Proxmox Host Script**: Shuts down the Proxmox host once all VMs have powered off.  
✅ **VM Script**: Shuts down a VM when it detects that the UPS-hosting VM is unreachable.  
✅ **Uninstallation Script**: Fully removes the scripts and services if no longer needed.  
✅ **Customizable Settings**: Modify parameters like ping frequency, failure threshold, and log locations.  

---

## 🚀 Installation  

### **1️⃣ Proxmox Host Script**  
📌 **What It Does**  
- Runs on the **Proxmox host**.  
- **Pings the UPS-hosting VM** to check if it is online.  
- If the UPS VM is **unreachable for a set number of attempts**:  
  - **Checks for running VMs**.  
  - If no VMs are running, **shuts down the Proxmox host**.  
  - If VMs are still running, it **waits** until they power off before shutting down.  

📜 **Installation Command (Run on Proxmox Host)**  
```bash
bash -c 'cat > /usr/local/bin/power-check.sh <<EOF
#!/bin/bash

LOGFILE="/var/log/power-check.log"
WINDOWS_VM_IP="192.168.50.218"
FAIL_COUNT=5
PING_INTERVAL=15
FAILED=0

are_vms_running() {
  RUNNING_VMS=\$(qm list | awk '\''$3 == "running" {print $1}'\'')
  if [[ -n "\$RUNNING_VMS" ]]; then
    echo "\$(date) - Proxmox VMs are still running. Retrying shutdown check..." >> "\$LOGFILE"
    return 0
  else
    return 1
  fi
}

> "\$LOGFILE"

while true; do
  if ping -c 1 -W 2 "\$WINDOWS_VM_IP" >/dev/null; then
    FAILED=0
  else
    ((FAILED++))
    echo "\$(date) - Ping failed (\$FAILED/\$FAIL_COUNT)" >> "\$LOGFILE"
  fi

  if [[ "\$FAILED" -ge "\$FAIL_COUNT" ]]; then
    while are_vms_running; do
      sleep "\$PING_INTERVAL"
    done
    echo "\$(date) - No response from Windows VM and no running VMs. Shutting down Proxmox..." >> "\$LOGFILE"
    shutdown -h +1
    exit 0
  fi

  sleep "\$PING_INTERVAL"
done
EOF

chmod +x /usr/local/bin/power-check.sh

cat > /etc/systemd/system/power-check.service <<EOF
[Unit]
Description=Shutdown Proxmox if Windows VM is unreachable
After=network.target

[Service]
ExecStart=/usr/local/bin/power-check.sh
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

systemctl enable power-check
systemctl start power-check'

 VM Script
📌 What It Does

Runs on a VM that should power down when the UPS-hosting VM goes offline.

Pings the UPS VM at intervals to check its status.

If pings fail a set number of times, the VM shuts itself down.

📜 Installation Command (Run on the VM)

bash
Copy
Edit
bash -c 'cat > /usr/local/bin/power-check.sh <<EOF
#!/bin/bash

# Clear the log file when the script starts
> /var/log/power-check.log

WINDOWS_VM_IP="192.168.50.218"
LOGFILE="/var/log/power-check.log"
FAIL_COUNT=5
PING_INTERVAL=15
FAILED=0

while true; do
  if ping -c 1 -W 2 \$WINDOWS_VM_IP >/dev/null; then
    FAILED=0
  else
    ((FAILED++))
    echo "\$(date) - Ping failed (\$FAILED/\$FAIL_COUNT)" >> \$LOGFILE
  fi

  if [ \$FAILED -ge \$FAIL_COUNT ]; then
    echo "\$(date) - No response from Windows VM. Shutting down..." >> \$LOGFILE
    shutdown -h +1
    exit 0
  fi

  sleep \$PING_INTERVAL
done
EOF'

chmod +x /usr/local/bin/power-check.sh

cat > /etc/systemd/system/power-check.service <<EOF
[Unit]
Description=Shutdown on Windows VM loss
After=network.target

[Service]
ExecStart=/usr/local/bin/power-check.sh
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

systemctl enable power-check
systemctl start power-check
❌ Uninstallation Script
📜 What It Does

Stops and disables the power-check service.

Kills any running instances of the script.

Removes script files, logs, and services.

📜 Run This Command to Remove Everything

bash
Copy
Edit
#!/bin/bash

# Stop and disable the service (if using systemd)
systemctl stop power-check 2>/dev/null
systemctl disable power-check 2>/dev/null
systemctl daemon-reload 2>/dev/null
systemctl reset-failed 2>/dev/null

# Stop the service using Synology's method (if applicable)
synoservice --stop power-check 2>/dev/null

# Kill any running instances
pkill -f power-check.sh 2>/dev/null

# Remove script, logs, and service files
rm -f /usr/local/bin/power-check.sh
rm -f /var/log/power-check.log
rm -f /etc/systemd/system/power-check.service
rm -f /usr/local/etc/rc.d/power-check.sh  # For DSM 6 and older

# Final cleanup and verification
systemctl daemon-reexec 2>/dev/null
echo "Power-check script and service fully removed!"
⚙️ Customization Options
You can modify the following variables in the script to fit your needs:

Variable	Default Value	Description
WINDOWS_VM_IP	192.168.50.218	IP address of the UPS-hosting VM to ping.
LOGFILE	/var/log/power-check.log	Path to store logs for monitoring ping failures and shutdown events.
FAIL_COUNT	5	Number of failed pings before initiating shutdown.
PING_INTERVAL	15 seconds	How often (in seconds) to ping the UPS VM.
FAILED	0	Counter for failed pings (do not change this in normal use).
🔹 To customize these settings, edit /usr/local/bin/power-check.sh and update the values as needed.

📜 Checking Logs
To view the logs for debugging or confirmation, run:

bash
Copy
Edit
cat /var/log/power-check.log
📜 License
This project is released under the MIT License.


