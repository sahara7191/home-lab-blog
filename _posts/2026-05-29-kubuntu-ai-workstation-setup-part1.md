---
title: "Kubuntu 26.04 LTS Fresh Setup: Installing and Configuring the OS (Part 1)"
date: 2026-05-29
categories:
  - homelab
  - ai
permalink: /homelab/kubuntu-ai-setup-part1/
tags:
  - kubuntu
  - linux
  - amd
  - rocm
  - bios
layout: single
author_profile: true
toc: true
---

After spending time exploring local AI tools on Windows, I decided to move my Corsair AI Workstation 300 to **Kubuntu 26.04 LTS** for better performance, full control over my data, and a more cybersecurity-friendly environment. This post covers the OS installation and base configuration. Part 2 covers the full AI stack: Ollama, Open WebUI, and n8n.

**Why Kubuntu instead of Ubuntu?** Kubuntu is an official Ubuntu flavour with the same base, same `apt`, and same everything underneath, but with the KDE Plasma desktop instead of GNOME. It feels more like Windows in layout, which makes the transition easier, and it is significantly lighter on resources, leaving more RAM for models.

---

## Hardware

```
system       Corsair AI Workstation 300
processor    AMD Ryzen AI Max+ 395 (16-core Zen 5, 32 threads)
gpu          AMD Radeon 8060S iGPU (gfx1151 / Strix Halo)
memory       128 GB LPDDR5X unified memory (up to 96 GB addressable as VRAM)
storage      4 TB (2 x 2 TB M.2 NVMe SSD)
os           Kubuntu 26.04 LTS (Resolute Raccoon)
kernel       7.0
```

> **Important:** The Radeon 8060S uses unified memory. There is no separate VRAM chip. The GPU and CPU share the same 128 GB pool, which is what allows running 70B+ models fully on-GPU — impossible on most consumer hardware.

---

## Prerequisites

**Kubuntu 26.04 LTS ISO**
Download from the official Kubuntu website:
[https://kubuntu.org/getkubuntu/](https://kubuntu.org/getkubuntu/)

**Ventoy USB (recommended)**
Ventoy lets you boot multiple ISOs from one USB drive without reformatting it each time. I recommend it over Rufus or BalenaEtcher for flexibility.
[https://www.ventoy.net/](https://www.ventoy.net/)

A USB drive with at least **8 GB** of space is required.

---

## Step 1 — Create Bootable USB with Ventoy

Download and install Ventoy on your USB drive (this will format it, so back up any existing files first). Once installed, copy the Kubuntu `.iso` file to the USB drive.

```
USB structure after setup:
├── ventoy/          <- Ventoy system files (do not touch)
└── kubuntu-26.04-desktop-amd64.iso  <- your ISO
```

---

## Step 2 — BIOS Configuration

Before booting from the USB, configure the BIOS first. This avoids a failed security check during installation and ensures the GPU memory is set up correctly for Linux from the start.

Restart and enter the BIOS/UEFI (press **Delete** during boot on the Corsair workstation).

### Disable Secure Boot

```
Security -> Secure Boot -> Secure Boot Enable -> Disabled
```

Secure Boot only allows Microsoft-signed bootloaders by default. Kubuntu's bootloader is signed but the chain of trust fails on AMD systems, especially with Ventoy. Disabling it is the standard approach for any Linux home lab setup.

> For a dedicated home lab workstation, leaving Secure Boot off is fine. Full disk encryption during installation provides equivalent protection for this use case.

### Configure iGPU Memory

The Corsair AI Workstation 300 uses a simplified BIOS that exposes only key settings with friendlier names than the full AMD CBS menu.

Navigate to:
```
Advanced -> GFX Configuration -> iGPU Configuration
```

You will see two options:

| Option | Choose? |
|---|---|
| AUTO | ***No*** — allocates 64 GB fixed carve-out automatically, leaving only ~62 GiB visible to the OS |
| **UMA_SPECIFIED** | ***Yes*** — lets you set the minimum carve-out manually |

Select **UMA_SPECIFIED** and enter **1024 MB (1 GB)** as the value.

This keeps the fixed GPU carve-out at 1 GB and lets the Linux kernel manage the remaining ~123 GB dynamically via GTT. The GPU can still access the full memory pool for model inference, but the OS sees the full 124 GiB instead of just 62 GiB.

> AUTO sounds like the right choice but in practice it reserves 64 GB as a fixed carve-out on this hardware, cutting visible RAM in half. UMA_SPECIFIED with 1 GB gives a much better result on Linux.

### Check remaining GFX settings

While in this menu, confirm these two settings are enabled (they should be by default):

 PCIe Resizable BAR -> *Enabled* 
 Above 4G Decoding -> *Enabled*


These two always go together. Resizable BAR allows the CPU to access the full GPU memory at once instead of in 256 MB chunks. Above 4G Decoding is required for Resizable BAR to work. Both are critical for LLM inference performance on unified memory hardware.

> There is no SDMA toggle in the Corsair BIOS. On Linux, SDMA is disabled via an environment variable in Step 6.

Save all settings and exit (`F10`).

---

## Step 3 — Install Kubuntu

### Before booting the installer

Connect an ethernet cable if possible. The installer can download updates and third-party drivers during installation, which saves time after first boot. Wi-Fi may not be available until after the OS is installed, as the driver is not always loaded in the live environment.

Plug in the Ventoy USB, restart, and press **F11** to enter the boot menu. Select the Ventoy USB device, then choose the Kubuntu ISO from the Ventoy menu.

### Installation options

- Language: English
- Keyboard: your preference
- Network: connect to ethernet or Wi-Fi if available
- Installation type: **Erase disk and install Kubuntu** (full fresh install)
- Filesystem: **ext4** (the default, leave it as is)
- Check both: **Download updates** and **Install third-party drivers**
- Create a strong username and password

> If you plan to dual-boot with Windows later, choose "Install Kubuntu alongside Windows" instead of erasing. For a dedicated AI workstation, a full erase gives cleaner performance.

> If you have two NVMe drives, the installer may only show one. Install on the first disk and set up the second drive manually after first boot.

The installation takes approximately 10-15 minutes. After completion, remove the USB and reboot.

---

## Step 4 — First Boot Configuration

After logging into KDE Plasma for the first time, open a terminal (`Ctrl+Alt+T` or search for Konsole in the application menu).

### Update the system

Always start with a full system update before installing anything else.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt dist-upgrade -y
sudo apt autoremove -y
```

### Install essential tools

```bash
sudo apt install -y \
  curl \
  wget \
  git \
  htop \
  fastfetch \
  nano \
  build-essential \
  software-properties-common \
  apt-transport-https \
  ca-certificates \
  gnupg \
  lsb-release
```

> `neofetch` was removed from Ubuntu repositories. Use `fastfetch` instead — it is the actively maintained replacement with the same system info output.

Verify your system with:

```bash
fastfetch
```

### Verify memory

Confirm the BIOS setting from Step 2 worked correctly:

```bash
free -h
```

On a 128 GB system you should see approximately **124 GiB** total. If you see ~62 GiB, go back into BIOS and confirm iGPU Configuration is set to UMA_SPECIFIED with 1024 MB.

### Fix swap

The Kubuntu installer creates a small swap partition by default. For a machine running large language models, this needs to be increased to at least 16 GB.

Check the current swap first:

```bash
cat /etc/fstab | grep swap
```

If a swapfile already exists, disable it and recreate it at 16 GB:

```bash
sudo swapoff /swapfile
sudo dd if=/dev/zero of=/swapfile bs=1G count=16 status=progress
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Make sure only one swap entry exists in fstab:

```bash
cat /etc/fstab | grep swap
# Should show exactly one line
```

If you see two lines, remove the duplicate:

```bash
sudo sed -i '/^\/swapfile none swap sw 0 0/d' /etc/fstab
```

Verify the final result:

```bash
free -h
```

Expected output:

```
Mem:   124Gi
Swap:   15Gi
```

---

## Step 5 — Install AMD ROCm

ROCm (Radeon Open Compute) is AMD's GPU compute platform. It allows Ollama and other AI tools to use the GPU for inference instead of falling back to slower CPU computation. Without ROCm, large models (30B+) run unacceptably slow.

One of the most welcome changes in Ubuntu 26.04 LTS is that ROCm now ships natively through Ubuntu's official repositories. No third-party repos, no GPG keys, no codename issues — a single `apt install` is all that's needed.

### Install ROCm

```bash
sudo apt install -y rocm
sudo systemctl daemon-reload
```

### Install monitoring tools

```bash
sudo apt install -y rocm-smi amd-smi
```

### Add your user to the render and video groups

This allows your user account (not just root) to access the GPU. Required for Ollama and other tools to use ROCm correctly.

```bash
sudo usermod -aG render,video $USER
sudo reboot
```

> Do not use `newgrp render` on KDE Plasma with Wayland. It interferes with the compositor's input handling and can freeze your mouse. A reboot applies the group change cleanly and permanently.

### Verify ROCm detects your GPU

```bash
rocminfo | grep gfx1151
```

Expected output:

```
Name:                    gfx1151
Name:                    amdgcn-amd-amdhsa--gfx1151
```

Also confirm your user is in the correct groups:

```bash
groups
# render and video should appear in the output
```

---

## Step 6 — Set GPU Override Variables

The Radeon 8060S uses GPU architecture `gfx1151`. Some ROCm builds do not recognize this identifier by default and fall back to CPU inference. Two environment variables fix this.

### Set them permanently system-wide

```bash
sudo nano /etc/environment
```

Add these lines at the bottom:

```
HSA_OVERRIDE_GFX_VERSION=11.5.1
HSA_ENABLE_SDMA=0
```

Save and exit (`Ctrl+X`, then `Y`, then `Enter`).

Apply immediately without rebooting:

```bash
source /etc/environment
echo $HSA_OVERRIDE_GFX_VERSION
# Should output: 11.5.1
```

> `HSA_OVERRIDE_GFX_VERSION=11.5.1` tells ROCm the correct GPU architecture. Without it, Ollama may silently fall back to CPU-only inference — roughly 2-5 tokens/sec on a 30B model instead of 30-40 tokens/sec.
>
> `HSA_ENABLE_SDMA=0` disables SDMA, which has a known bug on unified memory systems like Strix Halo that can cause inference instability.

---

## What's Next

With Kubuntu installed and ROCm configured, the next post covers the full AI stack: Ollama for local LLMs, Open WebUI for a ChatGPT-style interface, and n8n for agentic AI automation workflows.

[Continue to Part 2: Ollama, Open WebUI & n8n](/home-lab-blog/homelab/kubuntu-ai-setup-part2/)
