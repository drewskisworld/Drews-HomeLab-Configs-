# Drews-HomeLab-Configs

Technical documentation, secure configurations, and recovery procedures for a **Cisco Catalyst 3850 (WS-C3850-48P)** in a macOS-based home lab.

This repository documents how I recovered, wiped, and hardened a decommissioned enterprise switch with unknown credentials and prepared it for secure use in my home lab.

---

## Lab Environment

### Management Host

* **Machine:** Mac Mini (2014)
* **Operating System:** macOS

### Console Access Hardware

* **Console Cable:** USB-to-RJ45 (FTDI chipset)
* **Physical Connection:**

  * USB end → Mac Mini
  * RJ45 end → `CON` (Console) port on the rear of the switch

### Terminal Software

* **Tool:** macOS Native Terminal
* **Utility:** `screen`

---

## Accessing the Switch CLI (macOS)

First, identify the USB serial device created by the console cable:

```bash
ls /dev/tty.usb*
```

The device appeared as:

```text
/dev/tty.usbserial-XXXX
```

Start a console session using Cisco’s default baud rate:

```bash
screen /dev/tty.usbserial-XXXX 9600
```

At this point, the switch CLI is accessible.

---

## Project Overview

This Cisco Catalyst 3850 arrived with:

* Unknown credentials
* Existing enterprise configuration
* A **damaged external Mode button**

### Goals

* Regain access to the device
* Bypass the previous owner’s configuration
* Fully wipe system and VLAN data
* Apply a clean, secure baseline configuration for home lab use

---

## Phase 1: Hardware Recovery & Factory Bypass

### Mode Button Workaround

The external Mode button was damaged. Instead of replacing hardware, I used an **Allen key** to reach and press the internal tactile switch through the button opening.

### Physical Recovery Procedure

1. Powered the switch off
2. Inserted the Allen key into the Mode button opening
3. Held the internal button down
4. Powered the switch on while continuing to hold the button

### Visual Confirmation

* Front panel LEDs transitioned from **blinking green** to **solid amber**
* This confirmed entry into the **bootloader (`switch:`) environment**, bypassing the startup configuration

---

## Bootloader Commands

Once at the `switch:` prompt, I ran the following commands:

```text
flash_init
```

* Initializes the flash filesystem for file access.

```text
dir flash:
```

* Lists flash contents and revealed the existing `nvram_config`.

```text
delete flash:nvram_config
```

* Attempted to remove the old configuration.
* Failed due to the filesystem being **read-only**.

```text
SWITCH_IGNORE_STARTUP_CFG=1
```

* Instructs the switch to **ignore the startup configuration** during boot.

```text
boot
```

* Boots the switch while bypassing the previous owner’s config.

---

## Post-Boot Cleanup (Privileged EXEC Mode)

After booting into `Switch#` without authentication, I wiped all remaining configuration data.

```bash
write erase
```

* Clears the startup configuration.

```bash
delete flash:vlan.dat
```

* Removes the old VLAN database.

---

## Making the Reset Permanent (Critical Step)

To ensure the switch never prompts for old credentials again, I reset the configuration register and saved the clean state.

```bash
configure terminal
config-register 0x2102
exit
copy running-config startup-config
```

This guarantees a clean boot every time.

---

## Status After Recovery

* **Hostname:** Default (`Switch`)
* **Access Level:** Privileged EXEC (`#`) without a password
* **Filesystem:** Cleared of legacy configs and VLAN data

The switch was now fully reset and ready for configuration.

---

## Phase 2: Base Configuration & Hardening

### Step 1: Identity & Basic Security

Enter configuration mode:

```bash
enable
configure terminal
```

Set hostname:

```bash
hostname DrewsHomeSwitch
```

Set enable password:

```bash
enable secret <PASSWORD>
```

Encrypt all stored passwords:

```bash
service password-encryption
```

---

### Step 2: Management IP Configuration

Assign a management IP to VLAN 1:

```bash
interface vlan 1
 ip address 192.168.1.X 255.255.255.0
 no shutdown
 exit
```

Set default gateway:

```bash
ip default-gateway 192.168.1.X
```

This allows LAN-based management access.

---

### Step 3: Enable SSH (Secure Remote Access)

Set domain name (required for SSH):

```bash
ip domain-name drewshome.lab
```

Generate RSA keys:

```bash
crypto key generate rsa
```

Key size used:

```text
2048
```

Create a local admin user:

```bash
username admin privilege 15 secret <PASSWORD>
```

Restrict remote access to SSH only:

```bash
line vty 0 15
 login local
 transport input ssh
 exit
```

This fully disables Telnet.

---

### Step 4: Save the Configuration

Cisco switches store the running config in memory. Without saving, power loss wipes everything.

```bash
end
copy running-config startup-config
```

Press **Enter** to confirm.

---

## Verification: Connectivity Test

1. Connect Ethernet from **Mac Mini → Switch Port 1**
2. Connect Ethernet from **Router → Switch Port 2**
3. From the Mac terminal:

```bash
ping 192.168.1.X
```

Successful replies confirm:

* VLAN interface is up
* IP configuration is correct
* Switch is reachable on the network

---

## Final Notes

This process took the switch from:

* Unknown credentials
* Broken Mode button
* Enterprise leftovers

to:

* Fully wiped
* Secure
* SSH-only management
* Home-lab ready

Future updates may include:

* VLAN segmentation
* Port security
* STP tuning
* Backup and recovery workflows

