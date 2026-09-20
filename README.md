# Example_Installation_for_ARCH-LINUX

<br>

## Arch Linux + Btrfs + GNOME Installation & Rollback Guide

## Features
* UEFI Booting via GRUB.
* Btrfs Subvolume Layout (@, @home, @cache, @log, @snapshots).
* GNOME Desktop Environment.
* Audio & Bluetooth Setup using PipeWire and WirePlumber.
* Security Hardening using firewalld and AppArmor.
* Atomic System Rollbacks integrated with pacman and grub-btrfs.

<br>

## Prerequisites
* Target Disk: /dev/nvme0n1 (adjust drive naming accordingly for /dev/sda or /dev/sda1).
* Firmware Mode: UEFI.
* Active Internet Connection: Wi-Fi (via iwctl) or Ethernet.

<br>

## Disclaimer
* No Warranties: The scripts, commands, and instructions contained in this repository are provided without warranty of any kind, express or implied. Use them at your own risk.
* Risk of Data Loss: Modifying system partitions, installing custom kernel modules, or executing system-level commands can result in data loss or unbootable systems. Always back up important data before performing system modifications.
* Not Official Documentation: This is a personal installation workflow and is not officially affiliated with or endorsed by Arch Linux or any related software projects. Always refer to official documentation (e.g., the Arch Wiki) for authoritative guides.
* Hardware Variability: Commands and driver instructions may vary depending on your specific hardware configuration. Verify all hardware identifiers (e.g., drive paths like /dev/nvme0n1) before execution.
