# Power-Check Scripts for Proxmox and VMs

## Overview
This repository contains scripts designed for graceful shutdown of Proxmox hosts and virtual machines in case of a power failure detected by a UPS-hosting VM.

There are three main scripts:

1. **Proxmox Host Script**: Runs on the Proxmox host and shuts it down once all VMs are powered off.
2. **VM Script**: Runs on a VM and shuts it down when it detects that the UPS VM has gone offline.
3. **Uninstallation Script**: Removes all files and services related to these scripts.

---

## 1. Proxmox Host Script

### What It Does
This script runs on the **Proxmox host** and performs the following actions:

- Pings the **UPS-hosting VM** to check if it is still online.
- If the UPS VM is **unreachable** for a certain number of failed attempts:
  - **Checks if any VMs are still running**.
  - If **no VMs are running**, it will **shut down the Proxmox host**.
  - If VMs are still running, it waits until they are powered off before shutting down the host.

### Installation
To install the Proxmox Host script, run the provided commands on your Proxmox host. This will configure the script to start on boot via a **systemd service**.

---

## 2. VM Script

### What It Does
This script runs on a **VM** (which is connected to the UPS) and performs the following actions:

- Pings the **UPS-hosting VM** periodically to check if it is still online.
- If the **UPS-hosting VM becomes unreachable** (after a set number of failed pings):
  - The VM **shuts itself down** to prevent any further operations when the UPS is no longer available.

### Installation
To install the VM Script, run the provided commands on the VM. Like the Proxmox Host script, it will be set up to run automatically on boot via a **systemd service**.

**Customization Options**:
- Modify the script to set the IP address of the **UPS-hosting VM**, the number of **ping retries** before shutting down, and the **interval between pings**.

---

## 3. Uninstallation Script

### What It Does
This script is used to **remove all files and services** associated with the Power-Check scripts.

It will:

- **Stop and disable** the systemd services.
- **Kill any running instances** of the scripts.
- **Delete script files**, logs, and service files to fully clean up the system.

### Installation
To remove the Power-Check scripts and services from your system, simply run the provided uninstallation script. This will ensure that all associated files and configurations are safely removed.

---

## ⚙️ Customization Options for All Scripts

You can modify the following variables in the scripts to suit your environment:

| Variable        | Default Value          | Description |
|-----------------|------------------------|-------------|
| `WINDOWS_VM_IP` | `192.168.50.218`       | The IP address of the UPS-hosting VM to ping. |
| `LOGFILE`       | `/var/log/power-check.log` | Path to store logs for monitoring ping failures and shutdown events. |
| `FAIL_COUNT`    | `5`                    | Number of failed pings before initiating shutdown. |
| `PING_INTERVAL` | `15` seconds           | Interval in seconds between each ping to the UPS-hosting VM. |
| `FAILED`        | `0`                    | Counter for the number of failed pings (no need to change this). |

To customize these settings, simply edit the scripts directly on your Proxmox host or VM and update the variables as needed.

---

## 📜 Checking Logs  

To view logs for troubleshooting or confirmation, you can check the log file at:  
```bash
cat /var/log/power-check.log
