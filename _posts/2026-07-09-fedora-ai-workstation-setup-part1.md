---
title: "Fedora Workstation with AI Setup: Installing and Configuring the OS for Strix Halo (Part 1)"
date: 2026-07-09
categories:
  - homelab
  - ai
permalink: /homelab/fedora-ai-setup-part1/
tags:
  - fedora
  - linux
  - amd
  - rocm
  - strix-halo
  - bios
layout: single
author_profile: true
toc: true
---


This is my second attempt at building a Linux AI workstation on the Corsair AI Workstation 300. My first try was Kubuntu 26.04, and while the AI stack itself worked well, I ran into repeated system freezes: mouse and keyboard locking up and eventually a black screen on boot. After research, it turned out the instability came from a known firmware issue and missing kernel parameters for the Strix Halo platform, not from the distro itself.

This time I went with **Fedora Workstation**. The [amd-strix-halo-toolboxes project](https://github.com/kyuz0/amd-strix-halo-toolboxes) documents the stable configurations for this exact hardware, Fedora ships the newest kernels and firmware fastest, and *gfx1151* support comes compiled in. The [Strix Halo wiki](https://llm-tracker.info/_TOORG/Strix-Halo) is another excellent community resource that tracks driver and ROCm progress for this chip.

This post covers OS installation and base configuration, including the stability fixes from day one. Part 2 covers the full AI stack: Ollama, Open WebUI, and n8n.


## Hardware

```
system       Corsair AI Workstation 300
processor    AMD Ryzen AI Max+ 395 (16-core Zen 5, 32 threads)
gpu          AMD Radeon 8060S iGPU (gfx1151 / Strix Halo)
memory       128 GB LPDDR5X unified memory (up to 96 GB addressable as VRAM)
storage      4 TB (2 x 2 TB M.2 NVMe SSD)
os           Fedora Workstation 44
```

> **Important:** The Radeon 8060S uses unified memory. There is no separate VRAM chip. The GPU and CPU share the same 128 GB pool, which is what allows running 70B+ models fully on-GPU, something impossible on most consumer hardware.


## Prerequisites

**Fedora Workstation ISO (latest version)**
Download from the official Fedora website:
[https://fedoraproject.org/workstation/download](https://fedoraproject.org/workstation/download)

**Ventoy USB (my preference)**
Ventoy lets you boot multiple ISOs from one USB drive without reformatting it each time. I recommend it over Rufus or BalenaEtcher for flexibility.
[https://www.ventoy.net/](https://www.ventoy.net/)

A USB drive with at least **8 GB** of space is required.

---

## Step 1: Create Bootable USB with Ventoy

Download and install Ventoy on your USB drive (this will format it, so back up any existing files first). Once installed, copy the Fedora `.iso` file to the USB drive.

---

## Step 2: BIOS Configuration

Before booting from the USB, configure the BIOS first. This avoids a failed security check during installation and ensures the GPU memory is set up correctly for Linux from the start.

Restart and enter the BIOS/UEFI (press **Delete** during boot on the Corsair workstation).

### Disable Secure Boot

```
Security -> Secure Boot -> Secure Boot Enable -> Disabled
```

Secure Boot only allows Microsoft-signed bootloaders by default. The chain of trust often fails on AMD systems, especially with Ventoy. 

> For a dedicated home lab workstation, leaving Secure Boot off is fine. Full disk encryption during installation provides equivalent protection for this use case.

### Configure iGPU Memory

Navigate to:
```
Advanced -> GFX Configuration -> iGPU Configuration
```

You will see two options:

| Option | Select | Notes |
|---|---|---|
| AUTO | No | Allocates a 64 GB fixed carve-out automatically, leaving only ~62 GiB visible to the OS |
| UMA_SPECIFIED | **Yes** | Set to 1024 MB, gives the full ~124 GiB to the OS |

Select **UMA_SPECIFIED** and enter **1024 MB (1 GB)** as the value.

This keeps the fixed GPU carve-out at 1 GB and lets the Linux kernel manage the remaining memory dynamically via GTT. The GPU can still access the full memory pool for model inference, but the OS sees 124 GiB instead of 62 GiB.

### Check remaining GFX settings

Confirm these two settings are enabled (they should be by default):

| Setting | Option |
|---|---|
| PCIe Resizable BAR | Enabled |
| Above 4G Decoding | Enabled |

**Resizable BAR** allows the CPU to access the full GPU memory at once instead of in 256 MB chunks. **Above 4G Decoding** is required for Resizable BAR to work. Both are critical for LLM inference performance on unified memory hardware.

Save all settings and exit.

---

## Step 3: Install Fedora

### Before booting the installer

Plug in the Ventoy USB, restart, and press **F11** to enter the boot menu. Select the Ventoy USB device, then choose the Fedora ISO from the Ventoy menu.

Fedora boots into a live desktop first. Click **Install Fedora** to start the installer.
Installation destination: select the first NVMe drive, choose **Erase disk and install**.

> If you have two NVMe drives, the installer may only show one. Install on the first disk and set up the second drive manually later, after installation.

 After completion, remove the USB and reboot.

---

## Step 4: First Boot Configuration

Always start with a full system update before installing anything else.

```bash
sudo dnf upgrade -y
```

This may take a while on first run. Reboot after it finishes:

```bash
sudo reboot
```

### Check the firmware version

The Strix Halo community identified that the **linux-firmware-20251125** package breaks ROCm on this hardware and causes instability and crashes ([source](https://github.com/kyuz0/amd-strix-halo-toolboxes)). Check what you have:

```bash
rpm -q linux-firmware
```

If you see version 20251125, upgrade immediately:

```bash
sudo dnf upgrade -y linux-firmware
```

Any version newer than 20251125 is fine. This single broken firmware release was a likely cause of the freezes I experienced on my first Linux attempt.

### Install tools

```bash
sudo dnf install -y \
  curl \
  wget \
  git \
  htop \
  fastfetch \
  nano
```

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

### Set kernel parameters

This is the important step for stability, and the one I was missing on my first attempt. The Strix Halo community documented these kernel parameters as required for stable operation ([host configuration guide](https://github.com/kyuz0/amd-strix-halo-toolboxes)):

```bash
sudo grubby --update-kernel=ALL --args="amd_iommu=off amdgpu.gttsize=126976 ttm.pages_limit=32505856"
```

What each parameter does:

| Parameter | Purpose |
|---|---|
| amd_iommu=off | Disables the AMD IOMMU. Improves both performance and stability on Strix Halo |
| amdgpu.gttsize=126976 | Sets the GTT pool to ~124 GB so the GPU can use nearly all unified memory |
| ttm.pages_limit=32505856 | Raises the memory page limit to match the GTT size |

Reboot to apply:

```bash
sudo reboot
```

Verify the parameters are active:

```bash
cat /proc/cmdline
# Should include amd_iommu=off amdgpu.gttsize=126976 ttm.pages_limit=32505856
```
---

## Step 5: Install AMD ROCm

ROCm (Radeon Open Compute) is AMD's GPU compute platform. It allows Ollama and other AI tools to use the GPU for inference instead of falling back to slower CPU computation. Without ROCm, large models (30B+) run unacceptably slow.

Fedora ships ROCm natively in its repositories with gfx1151 support compiled in, one of the reasons the Strix Halo community prefers it.

### Install ROCm

```bash
sudo dnf install -y rocminfo rocm-smi rocm-hip rocm-runtime rocm-clinfo
```

### Add your user to the render and video groups

This allows your user account (not just root) to access the GPU. Required for Ollama and other tools to use ROCm correctly.

```bash
sudo usermod -aG render,video $USER
sudo reboot
```

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

## Step 6: Set GPU Override Variables

The Radeon 8060S uses GPU architecture `gfx1151`. Some ROCm builds do not recognize this identifier by default and fall back to CPU inference. Two environment variables fix this.

### Set them permanently system-wide

```bash
sudo nano /etc/environment
```

Add these lines:

```
HSA_OVERRIDE_GFX_VERSION=11.5.1
HSA_ENABLE_SDMA=0
```

Save and exit.

Apply immediately without rebooting:

```bash
source /etc/environment
echo $HSA_OVERRIDE_GFX_VERSION
# Should output: 11.5.1
```

> `HSA_OVERRIDE_GFX_VERSION=11.5.1` tells ROCm the correct GPU architecture. Without it, Ollama may silently fall back to CPU-only inference, roughly 2-5 tokens/sec on a 30B model instead of 30-40 tokens/sec.
>
> `HSA_ENABLE_SDMA=0` disables SDMA, which has a known bug on unified memory systems like Strix Halo that can cause inference instability.

---

### Community references:
- [amd-strix-halo-toolboxes on GitHub](https://github.com/kyuz0/amd-strix-halo-toolboxes) with the [interactive benchmark viewer](https://kyuz0.github.io/amd-strix-halo-toolboxes/)
- [Strix Halo wiki on llm-tracker.info](https://llm-tracker.info/_TOORG/Strix-Halo)

---

## What's Next

With Fedora installed, kernel parameters set, and ROCm configured, the next post covers the full AI stack: Ollama for local LLMs, Open WebUI for a ChatGPT-style interface, and n8n for agentic AI automation workflows.

[Continue to Part 2: Ollama, Open WebUI & n8n](/home-lab-blog/homelab/fedora-ai-setup-part2/)
