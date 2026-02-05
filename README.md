# How-To-Create-Home-Server
### Converting an Old PC to a Secure Home Server for Node.js APIs

This README documents the complete process of converting an old PC (Intel Pentium G2020 @ 2.90GHz, 4GB RAM, 250GB HDD) into a secure home server running Ubuntu Server 24.04 LTS. The goal was to host Node.js APIs with permanent endpoints, future FTP support, external access, home network protection, and automated power management for cooling/rest. The process involved troubleshooting hardware issues, installation challenges, network fixes, security hardening, and features like Tailscale for CGNAT bypass. All steps are sequential, including errors and solutions.

## Project Overview
- **Hardware Specs**:
  - CPU: Intel(R) Pentium(R) CPU G2020 @ 2.90GHz
  - RAM: 4GB
  - Storage: 250GB HDD
  - Motherboard: Gigabyte H61M-S2P (UEFI DualBIOS, F6 version)
  - Network: Realtek RTL8111/8168/8411 onboard Ethernet
  - Router: ZTE ZXHN H108N V2.5 (TE Data ISP, CGNAT)
- **OS**: Ubuntu Server 24.04 LTS (Noble Numbat)
- **Goals**: Secure server for Node.js APIs, remote access, power scheduling (shutdown at 2 AM/3 PM Egypt time, wake at 7 AM/5 PM), home network isolation.
- **Challenges**: Hardware boot loops, network driver failures, installer crashes, SSH port issues, CGNAT blocking forwarding, RTC wake failures.
- **Tools/Services**: Ubuntu Server, Netplan, UFW, Fail2Ban, Node.js, PM2, Nginx, Certbot, Tailscale, DuckDNS, rtcwake/cron for scheduling, WoL script.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Hardware Troubleshooting](#hardware-troubleshooting)
3. [Ubuntu Server Installation](#ubuntu-server-installation)
4. [Post-Install Configuration](#post-install-configuration)
5. [Network Setup and Fixes](#network-setup-and-fixes)
6. [Security Hardening](#security-hardening)
7. [SSH Key Setup and Remote Access](#ssh-key-setup-and-remote-access)
8. [External Access and CGNAT Bypass](#external-access-and-cgnat-bypass)
9. [Node.js and API Hosting](#nodejs-and-api-hosting)
10. [Power Scheduling and Wake Features](#power-scheduling-and-wake-features)
11. [WoL Setup for Manual Wakes](#wol-setup-for-manual-wakes)
12. [TimeZone and RTC Configuration](#timezone-and-rtc-configuration)
13. [Additional Features and Best Practices](#additional-features-and-best-practices)
14. [Lessons Learned and Recommendations](#lessons-learned-and-recommendations)

## Prerequisites
- USB flash drive (4GB+) for Ubuntu Server ISO.
- SATA-to-USB adapter (used for external GPT conversion).
- Backup any HDD data — install wipes it.
- Router admin access (TE Data defaults: user/user or admin/admin).
- Tailscale account (free) for remote access.
- BIOS access key: Del during boot.
- Knowledge: Basic CLI; developer-level for Node.js.

## Hardware Troubleshooting
### Issue: PC Powers On But No Monitor Signal, Restarts in Loop After Unallocating HDD
- Symptom: Fans spin, but no display; restarts after 10 seconds.
- Cause: No bootable OS, POST failure.
- Solution: Reseat RAM, clean contacts, clear CMOS (remove battery 5-10 min or short jumper), boot from Ubuntu USB. Fixed by minimal config test.

### Issue: No Signal to Monitor Initially
- Cause: Loose connections, RAM issues.
- Solution: Reseat cables/RAM, clear CMOS. Worked after.

### BIOS Settings for Power Management (Applied Later)
- Power Management: ErP Disabled, AC BACK Always On, Resume by Alarm Enabled (for daily wakes).
- Peripherals: LAN PXE Boot Option ROM Enabled (for WoL).
- M.I.T.: All Auto (no overclock).
- BIOS Features: CSM Support Enabled, Intel Virtualization Disabled.

## Ubuntu Server Installation
1. Create bootable USB with Rufus.
2. Boot: "Try or Install Ubuntu Server".
3. Language/Keyboard: English.
4. Network: Autoconfig failed — continued without.
5. Proxy: Blank.
6. Mirror: Default Egypt mirror.
7. Storage: Custom layout (pre-partitioned externally with GParted due to crashes).
8. Profile: Set name, hostname (e.g., "error404-nohope"), username (e.g., "sanfur602"), strong password.
9. SSH: Enabled.
10. Snaps: None.
11. Install completed.

### Issue: Installer Crashes on Partitioning
- Cause: Subiquity bug when deleting EFI or handling unallocated space.
- Solution: Used Ubuntu Desktop live USB + GParted to pre-partition (GPT table, EFI 1GB fat32, / 50GB ext4, /var 20GB ext4, /home 150GB ext4, swap 8GB). Then assigned mounts in Server installer.

## Post-Install Configuration
1. Log in locally.
2. Update: `sudo apt update && sudo apt full-upgrade -y`.
3. Install htop: `sudo apt install htop -y`.
4. Ubuntu Pro: Free for personal use — skipped.

## Network Setup and Fixes
- Issue: Ethernet autoconfig failed.
- Solution: Installed `r8168-dkms`, blacklisted `r8169` in `/etc/modprobe.d/blacklist-realtek.conf`, updated initramfs, rebooted.
- Static IP: Created `/etc/netplan/00-installer-config.yaml` with 192.168.1.100/24, applied `sudo netplan apply`.
- Issue: Permissions warnings on Netplan file.
- Solution: `sudo chmod 0600 /etc/netplan/00-installer-config.yaml`.

## Security Hardening
- SSH: Set Port 2222, PermitRootLogin no, PasswordAuthentication no (after keys).
- Firewall: `sudo apt install ufw -y`, allowed 2222/tcp, enabled.
- Fail2Ban: Installed.
- Unattended Upgrades: Enabled.
- Sysctl: Added security params in `/etc/sysctl.conf`, applied `sudo sysctl -p`.

### Issue: SSH Not Listening on 2222 After Reboot
- Cause: Systemd socket activation overriding port.
- Solution: Disabled ssh.socket, enabled ssh.service.

## SSH Key Setup and Remote Access
- Generated keys on Windows: `ssh-keygen -t ed25519`.
- Copied public key to server.
- Issue: Generated on server by mistake.
- Solution: Deleted `rm ~/.ssh/id_ed25519*`.

## External Access and CGNAT Bypass
- Issue: TE Data CGNAT (WAN IP 100.x.x.x, public IP 156.x.x.x) blocks forwarding.
- Solution: Tailscale for secure VPN tunnel.
  - Install: `curl -fsSL https://tailscale.com/install.sh | sh`.
  - Up: `sudo tailscale up`.
  - Enable on boot: `sudo systemctl enable tailscaled`.
- Access via Tailscale IP (e.g., 100.113.34.33).
- DuckDNS: Recommended for domain, but skipped due to CGNAT; use with Tailscale if needed.

## Node.js and API Hosting
- Install Node 20 LTS: Nodesource script.
- PM2: `sudo npm i -g pm2`, `pm2 startup`.
- Nginx: Installed, configured reverse proxy in `/etc/nginx/sites-available/default`.
- HTTPS: Certbot with DuckDNS domain.
- UFW: Allowed 'Nginx Full'.

## Power Scheduling and Wake Features
- Shutdowns: Cron `0 2 * * * /sbin/shutdown -h now` (2 AM Egypt).
- Wakes: rtcwake set before shutdown.
- Issue: rtcwake Not Waking from Off.
- Cause: ErP Enabled, HPET Interference, ACPI Events, Local RTC TZ.
- Solutions: Disabled ErP in BIOS, set RTC to UTC (`sudo timedatectl set-local-rtc 0`), disabled ACPI devices (EHC1/EHC2 in /proc/acpi/wakeup), added kernel params (`hpet=disable`, `acpi=force` in GRUB, then removed `acpi=force` due to reboot/shutdown mix-up).

### Issue: Reboot Acts as Shutdown
- Cause: `acpi=force` parameter.
- Solution: Removed from GRUB, updated.

- Timezone: Set to Africa/Cairo (`sudo timedatectl set-timezone Africa/Cairo`).

## WoL Setup for Manual Wakes
- BIOS: Enabled LAN PXE Boot Option ROM.
- Script (PowerShell on Windows):
  ```
  $mac = "90:2B:34:7C:7C:FA" -replace "[:-]",""
  $macBytes = for ($i = 0; $i -lt $mac.Length; $i += 2) {
      [byte]::Parse($mac.Substring($i,2), "HexNumber")
  }
  $packet = ([byte[]](,0xFF*6)) + ($macBytes * 16)
  $udp = New-Object System.Net.Sockets.UdpClient
  $udp.Send($packet, $packet.Length, "156.210.61.18", 9) | Out-Null
  $udp.Close()
  ```
- Router: Forward UDP 9 to 192.168.1.255 (broadcast).
- Issue: WoL Not Working Externally.
- Cause: CGNAT, router broadcast limit.
- Solution: Tailscale relay or public IP from ISP.

## TimeZone and RTC Configuration
- Set timezone to Africa/Cairo.
- RTC in local TZ: Set to no (UTC) for rtcwake compatibility.

## Additional Features and Best Practices
- Monitoring: htop.
- Backups: rsync to external.
- Ubuntu Pro: Free for personal use.
- rc.local: Enabled via systemd service for persistent ACPI disables.

## Lessons Learned and Recommendations
- Use GParted for partitioning to avoid installer bugs.
- Tailscale is essential for CGNAT ISPs like TE Data.
- Disable socket activation for custom SSH ports.
- For rtcwake failures, prioritize BIOS alarm or WoL.
- Upgrade RAM/SSD if performance lags.
- Always test short intervals for scheduling.

This setup is now secure, scheduled, and accessible via Tailscale. For future expansions (FTP: vsftpd with chroot), follow similar hardening steps. Happy hosting!
