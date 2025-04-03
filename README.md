Power-Check Scripts for Proxmox and VMs
Overview
This repository contains scripts designed for graceful shutdown of Proxmox hosts and virtual machines in case of a power failure detected by a UPS-hosting VM.

There are two main scripts:

Proxmox Host Script: Runs on the Proxmox host and shuts it down once all VMs are powered off.

VM Script: Runs on a VM and shuts it down when it detects that the UPS VM has gone offline.

Additionally, there is an uninstallation script that removes all files and services related to these scripts.

1. Proxmox Host Script
What It Does
This script runs on a Proxmox host.

It pings the UPS-hosting VM to check if it is still online.

If the UPS VM is unreachable for a certain number of failed attempts:

It checks if any other VMs are still running.

If no VMs are running, the Proxmox host shuts down.

If VMs are still running, it waits until they are powered off before shutting down the host.

Installation on Proxmox Host
Run the following command on your Proxmox host to install and enable the script:

bash
Copy
Edit
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
2. VM Script
What It Does
This script runs on a VM that needs to be shut down when the UPS-hosting VM goes offline.

It pings the UPS VM regularly.

If the UPS VM does not respond after 5 failed attempts, this script shuts down the VM.

Installation on the VM
Run the following command on the VM to install and enable the script:

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
3. Uninstallation Script
What It Does
Stops and disables the power-check service.

Kills any running instances of the script.

Removes all script and log files.

Run This Script to Remove Everything
To uninstall the power-check scripts and service, run the following command:

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
4. Customization Options
You can modify the following variables in the script to fit your needs:

Variable	Default Value	Description
WINDOWS_VM_IP	192.168.50.218	IP address of the UPS-hosting VM to ping.
LOGFILE	/var/log/power-check.log	Path to store logs for monitoring ping failures and shutdown events.
FAIL_COUNT	5	Number of failed pings before initiating shutdown.
PING_INTERVAL	15 seconds	How often (in seconds) to ping the UPS VM.
FAILED	0	Counter for failed pings (do not change this in normal use).
To customize these settings, edit /usr/local/bin/power-check.sh and change the values as needed.

5. Checking Logs
If you want to check the logs for debugging or confirmation, run:

bash
Copy
Edit
cat /var/log/power-check.log
6. License
This project is released under the MIT License.
