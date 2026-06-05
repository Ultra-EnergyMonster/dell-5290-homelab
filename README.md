# Dell Latitude 5290 Homelab Project

This repository documents how I repurposed a **Dell Latitude 5290** into a low-power homelab machine for self-hosting, automation, and local AI experimentation.

## Project Summary
- Converted an old business laptop into a 24/7 home server
- Tested **LacteraOS** installation and fallback strategies
- Brought up **Home Assistant** for local smart-home automation
- Evaluated reliability, thermals, and storage constraints for long-running workloads

## Hardware
- **Model:** Dell Latitude 5290
- **CPU:** Intel i5 (U-series)
- **RAM:** 16 GB DDR4
- **Storage:** 512 GB NVMe SSD
- **Network:** Gigabit Ethernet (USB adapter) + Wi-Fi
- **Power Strategy:** Kept on UPS-backed outlet for short outage tolerance

## Why This Project
I wanted to show practical homelab skills on real, constrained hardware:
- OS installation and recovery
- service deployment and troubleshooting
- networking and local DNS organization
- documenting failures and lessons learned

## LacteraOS Installation Attempts
I started with LacteraOS because I wanted a lightweight host that could run containers and stay responsive on laptop hardware.

### Attempt 1 — Direct USB Install
1. Flashed ISO with Balena Etcher
2. Booted in UEFI mode
3. Installer launched, but disk detection was inconsistent

**Result:** Failed install due to storage controller mismatch during partition stage.

### Attempt 2 — BIOS/UEFI and Storage Mode Tuning
1. Updated BIOS
2. Switched storage mode and secure boot settings
3. Recreated installation media

**Result:** Installer completed, but bootloader failed after first reboot.

### Attempt 3 — Clean Repartition + Manual Boot Repair
1. Wiped drive and recreated GPT layout
2. Reinstalled LacteraOS
3. Repaired EFI entries manually

**Result:** System booted successfully and ran stable enough for service tests.

## Home Assistant Deployment
After getting a stable base system, I tested Home Assistant in a containerized setup.

### What I Set Up
- Home Assistant Core container
- Host networking for local device discovery
- Persistent config volume on NVMe
- Auto-restart policy for resilience

### Issues I Hit
- mDNS discovery initially unreliable across VLAN boundaries
- One Zigbee bridge had intermittent reconnect loops
- Startup delays when other containers were saturating disk I/O

### Fixes
- Adjusted network topology and multicast settings
- Moved integrations to static host mappings where possible
- Added health-check and restart guards for unstable containers

## Other Services Tested
- Local reverse proxy for internal services
- Lightweight metrics/log monitoring
- Initial local AI runtime tests (resource-constrained)

## Outcome
The Dell 5290 now works as a practical homelab node for:
- home automation
- always-on internal services
- controlled experiments with self-hosted tools

## Key Skills Demonstrated
- Linux installation, recovery, and boot troubleshooting
- Container deployment and service hardening
- Network-level troubleshooting for local discovery protocols
- Incremental documentation and technical communication

## Next Steps
- Add infrastructure-as-code for reproducible setup
- Expand observability stack
- Benchmark and tune local AI workloads further
