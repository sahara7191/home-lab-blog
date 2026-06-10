---
title: "Kubuntu 26.04 LTS Fresh Setup: AI Workstation with Ollama, Docker, Open WebUI & n8n"
date: 2026-05-29
categories:
  - homelab
  - ai
permalink: /homelab/kubuntu-ai-setup/
tags:
  - kubuntu
  - linux
  - ollama
  - docker
  - n8n
  - amd
  - rocm
  - llm
  - privacy
  - open-webui
layout: single
author_profile: true
toc: true
---

After spending time exploring local AI tools on Windows, I decided to move my Corsair AI Workstation 300 to **Kubuntu 26.04 LTS** for better performance, full control over my data, and a more cybersecurity-friendly environment. This post documents every step of my fresh setup, from OS installation to running local LLMs entirely offline with Ollama, Docker, Open WebUI, and n8n.

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

|---|---|
| AUTO | Allocates 64 GB fixed carve-out automatically, leaving only ~62 GiB visible to the OS |
| **UMA_SPECIFIED** | Lets you set the minimum carve-out manually |

Select **UMA_SPECIFIED** and enter **1024 MB (1 GB)** as the value.

This keeps the fixed GPU carve-out at 1 GB and lets the Linux kernel manage the remaining ~123 GB dynamically via GTT. The GPU can still access the full memory pool for model inference, but the OS sees the full 124 GiB instead of just 62 GiB.

> AUTO sounds like the right choice but in practice it reserves 64 GB as a fixed carve-out on this hardware, cutting visible RAM in half. UMA_SPECIFIED with 1 GB gives a much better result on Linux.

### Check remaining GFX settings

While in this menu, confirm these two settings are enabled (they should be by default):

| Setting | Should be |
|---|---|
| PCIe Resizable BAR | Enabled |
| Above 4G Decoding | Enabled |

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
  neofetch \
  nano \
  build-essential \
  software-properties-common \
  apt-transport-https \
  ca-certificates \
  gnupg \
  lsb-release
```

> `build-essential` is required for compiling packages later. `gnupg` and `ca-certificates` are needed for securely adding third-party repositories.

Verify your system with:

```bash
neofetch
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

### Add the AMD ROCm repository

```bash
# Download and add AMD's signing key
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | \
  gpg --dearmor | sudo tee /etc/apt/keyrings/rocm.gpg > /dev/null

# Add the ROCm repository for Ubuntu 26.04
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] \
  https://repo.radeon.com/rocm/apt/7.2/ resolute main" | \
  sudo tee /etc/apt/sources.list.d/rocm.list

# Add the amdgpu repository
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] \
  https://repo.radeon.com/amdgpu/latest/ubuntu resolute main" | \
  sudo tee /etc/apt/sources.list.d/amdgpu.list
```

### Install ROCm packages

```bash
sudo apt update
sudo apt install -y rocm-hip-libraries rocm-dev rocminfo rocm-smi-lib
```

### Add your user to the render and video groups

This allows your user account (not just root) to access the GPU.

```bash
sudo usermod -aG render,video $USER
```

Log out and back in for group changes to take effect, or run:

```bash
newgrp render
```

### Verify ROCm detects your GPU

```bash
rocminfo | grep -A 5 "Agent 2"
```

You should see output referencing `gfx1151`, your GPU's architecture identifier.

---

## Step 6 — Critical: Set GPU Override Variable

The Radeon 8060S in the Ryzen AI Max+ 395 uses GPU architecture `gfx1151`. Some ROCm builds do not recognize this identifier by default and fall back to CPU inference. The fix is a single environment variable.

### Set it permanently system-wide

```bash
sudo nano /etc/environment
```

Add this line at the bottom:

```
HSA_OVERRIDE_GFX_VERSION=11.5.1
```

Save and exit (`Ctrl+X`, then `Y`, then `Enter`).

Apply immediately without rebooting:

```bash
source /etc/environment
echo $HSA_OVERRIDE_GFX_VERSION
# Should output: 11.5.1
```

> **Why this matters:** Without this variable, Ollama may silently fall back to CPU-only inference, giving you roughly 2-5 tokens/sec on a 30B model instead of 30-40 tokens/sec. This single line makes an enormous difference.

---

## Step 7 — Install Ollama

Ollama manages and runs local LLMs. It handles model downloads, memory allocation, and exposes an API at `localhost:11434` that Open WebUI and n8n connect to.

### Install via official script

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### Verify Ollama is running

```bash
ollama --version
systemctl status ollama
```

The service should show as `active (running)`.

### Configure Ollama to use ROCm

Create a systemd override to pass the GPU variable to the Ollama service:

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo nano /etc/systemd/system/ollama.service.d/override.conf
```

Add the following content:

```ini
[Service]
Environment="HSA_OVERRIDE_GFX_VERSION=11.5.1"
Environment="OLLAMA_HOST=0.0.0.0"
```

> `OLLAMA_HOST=0.0.0.0` makes Ollama accessible from other devices on your local network, useful for connecting from a laptop or phone via Open WebUI. Remove this line if you want it strictly local only.

Reload and restart the service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### Pull your first model

```bash
# Fast and capable — good starting point
ollama pull qwen3:30b

# For coding and script writing
ollama pull qwen2.5-coder:32b

# Quick test model
ollama pull llama3.2:3b
```

### Test inference and verify GPU is being used

```bash
ollama run llama3.2:3b "Hello, are you running on GPU?"
```

While it is running, open a second terminal and check GPU utilization:

```bash
rocm-smi
```

You should see GPU memory usage increasing as the model loads. If GPU memory stays at 0, the `HSA_OVERRIDE_GFX_VERSION` variable was not applied correctly to the service.

---

## Step 8 — Install Docker

Docker is required for running Open WebUI and n8n as containers. Containers keep these services isolated, easy to update, and independent of your system Python or Node versions.

### Add Docker's official repository

```bash
# Add Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker Engine

```bash
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

### Add your user to the docker group

This lets you run Docker commands without `sudo` every time.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Verify Docker is working

```bash
docker --version
docker run hello-world
```

You should see the "Hello from Docker!" message confirming everything is working.

---

## Step 9 — Install Open WebUI

Open WebUI is a ChatGPT-style web interface that connects to your local Ollama instance. It runs in Docker and lets you chat with any model you have pulled, manage conversations, and switch between models mid-chat.

### Create a Docker volume for persistent data

```bash
docker volume create open-webui
```

### Run Open WebUI

```bash
docker run -d \
  --name open-webui \
  --network=host \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://localhost:11434 \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

> `--network=host` allows the container to reach Ollama directly on `localhost:11434` without any additional networking configuration. `--restart always` means Open WebUI starts automatically on every boot.

### Access Open WebUI

Open your browser and navigate to:

```
http://localhost:8080
```

On first launch, create an admin account. Your username and password are stored locally. Nothing is sent to any external service.

You should immediately see your Ollama models listed in the model selector. Select one and start chatting — this is now 100% local, zero data leaving your machine.

---

## Step 10 — Install n8n (Self-Hosted)

n8n is a workflow automation platform. Self-hosting it means all your workflow data, credentials, and execution logs stay on your machine. Combined with the Anthropic API key (instead of claude.ai), this gives you a private AI automation stack.

### Create a dedicated directory for n8n

```bash
mkdir -p ~/n8n-data
```

### Run n8n with Docker

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v ~/n8n-data:/home/node/.n8n \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=admin \
  -e N8N_BASIC_AUTH_PASSWORD=changeme \
  --restart always \
  docker.n8n.io/n8nio/n8n
```

> Change `changeme` to a strong password before deploying. Basic auth protects the n8n UI from unauthorized access, especially if you later expose it on your local network.

### Access n8n

Open your browser and navigate to:

```
http://localhost:5678
```

Log in with the credentials you set above.

### Connect n8n to Ollama (local AI, zero cost)

Inside n8n, create a new workflow and add an **Ollama** node. Set the base URL to:

```
http://localhost:11434
```

This routes all AI requests through your local Ollama instance. No API key needed, no usage costs, no data leaving your machine.

### Connect n8n to Anthropic API (optional, for Claude)

For tasks where you need Claude's capabilities (better reasoning, larger context), add your Anthropic API key in n8n:

```
Settings -> Credentials -> New -> Anthropic API
```

Paste your API key from `console.anthropic.com`. When using this credential, prompts are sent to Anthropic's servers (7-day retention, never used for training) and results come back to your local n8n instance. No data is stored in claude.ai.

---

## Step 11 — Auto-start Everything on Boot

All three services are already configured with `--restart always` in Docker, and Ollama runs as a systemd service. Verify everything starts automatically:

```bash
# Check all running containers
docker ps

# Check Ollama service
systemctl status ollama
```

After a reboot, everything should come back up automatically within about 30 seconds.

---

## Step 12 — Verify the Full Stack

After rebooting, run the following checklist:

```bash
# 1. Check GPU recognition
rocminfo | grep gfx1151

# 2. Check Ollama is running and GPU-accelerated
ollama ps

# 3. Check Docker containers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Expected output:

```
NAMES          STATUS          PORTS
open-webui     Up X minutes
n8n            Up X minutes    0.0.0.0:5678->5678/tcp
```

Then confirm the web interfaces:

| Service     | URL                    |
|-------------|------------------------|
| Open WebUI  | http://localhost:8080  |
| n8n         | http://localhost:5678  |
| Ollama API  | http://localhost:11434 |

---

## Recommended Models to Pull

Given the 96 GB VRAM available on this machine, you can run models that are impossible on typical consumer hardware:

```bash
# Best all-round model, near cloud quality, 100% local
ollama pull qwen3:30b

# Best for coding and writing security scripts
ollama pull qwen2.5-coder:32b

# Deep reasoning, useful for log analysis and threat hunting
ollama pull deepseek-r1:32b

# Rivals cloud AI performance
ollama pull llama3.3:70b

# Ultimate, only possible with 96 GB VRAM
ollama pull qwen3:235b
```

> **Tip:** Start with `qwen3:30b`. It loads fast (~18 GB) and performs surprisingly close to 70B models for most tasks. Use it as your daily driver and pull larger models when you need deeper reasoning.

---

## Privacy Summary

This setup achieves the following data sovereignty:

| Component                | Data location       | Internet traffic                                              |
|--------------------------|---------------------|---------------------------------------------------------------|
| Ollama models            | Local disk          | None after initial download                                   |
| Open WebUI               | Local Docker volume | None                                                          |
| n8n workflows            | ~/n8n-data/         | None                                                          |
| Conversation history     | Local only          | None                                                          |
| Anthropic API (optional) | Local only          | Prompt and response only, 7-day log retention, never trained on |

Everything runs on your hardware. No conversation history is stored in any cloud service.

---

## Troubleshooting

**Ollama not using GPU (slow inference)**

Check that the environment variable is set correctly:
```bash
sudo systemctl cat ollama | grep HSA
# Should show: Environment="HSA_OVERRIDE_GFX_VERSION=11.5.1"
```

If missing, repeat Step 6 and restart the service.

**Open WebUI not connecting to Ollama**

Verify Ollama is listening:
```bash
curl http://localhost:11434/api/tags
```

If it returns JSON with your models, Ollama is working. Restart the Open WebUI container:
```bash
docker restart open-webui
```

**n8n container not starting**

Check logs for errors:
```bash
docker logs n8n
```

Permission issues with the data directory are the most common cause:
```bash
sudo chown -R 1000:1000 ~/n8n-data
docker restart n8n
```

---

## What's Next

With this stack running, the next steps in my home lab journey will be:

- Building n8n agentic AI workflows for automated threat intelligence gathering
- Connecting n8n to local security tools (VirusTotal API, AbuseIPDB)
- Running the Anthropic research agent cookbook locally for automated report writing
- Integrating Ollama models into CTF challenge analysis workflows
