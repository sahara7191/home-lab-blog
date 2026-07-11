---
title: "Fedora Workstation 43 AI Stack: Ollama, Open WebUI & n8n (Part 2)"
date: 2026-07-09
categories:
  - homelab
  - ai
permalink: /homelab/fedora-ai-setup-part2/
tags:
  - fedora
  - ollama
  - docker
  - n8n
  - open-webui
  - llm
  - privacy
layout: single
author_profile: true
toc: true
---
<br>

This is Part 2 of the Fedora Workstation 43 AI Workstation setup on the Corsair AI Workstation 300 (Strix Halo). Part 1 covered OS installation, BIOS configuration, the critical kernel parameters for stability, ROCm, and the GPU override variables. This post covers the full AI stack: Ollama for local LLMs, Open WebUI for a browser-based chat interface, and n8n for agentic AI automation.

[Back to Part 1: OS Installation and ROCm](/home-lab-blog/homelab/fedora-ai-setup-part1/)

---

## Step 7: Set Up the Second Drive (Optional)

The Corsair AI Workstation 300 ships with two 2 TB NVMe drives. Fedora installs onto the first one, leaving the second unformatted (it arrives with a leftover Windows BitLocker partition from the factory image). This second drive is ideal for storing large model files, disk images, memory dumps, and case data.

First, identify the drives:

```bash
sahara@fedora:~$ lsblk

NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
zram0       251:0    0    8G  0 disk [SWAP]
nvme0n1     259:0    0  1.9T  0 disk 
└─nvme0n1p1 259:2    0  1.9T  0 part 
nvme1n1     259:1    0  1.9T  0 disk 
├─nvme1n1p1 259:3    0  600M  0 part /boot/efi
├─nvme1n1p2 259:4    0    2G  0 part /boot
└─nvme1n1p3 259:5    0  1.9T  0 part /home
                                     /
```

You will see the system drive with `/boot`, `/`, and `/home` partitions, and a second drive with a single unused partition. In my case the system was on `nvme1n1` and the empty second drive was `nvme0n1`. Adjust the device names below to match your output.

### Format the second drive

```bash
sudo mkfs.ext4 /dev/nvme0n1p1
```

> The installer may warn that the partition contains a BitLocker filesystem labelled something like `CORSAIRAI D:`. This is the leftover factory Windows partition. If you have nothing on it to keep (a fresh workstation will not), type `y` to proceed.

### Mount it and make it permanent

```bash
sudo mkdir -p /mnt/storage
sudo mount /dev/nvme0n1p1 /mnt/storage
sudo blkid /dev/nvme0n1p1
```

Copy the UUID from the `blkid` output and add it to `fstab` so the drive mounts automatically on every boot:

```bash
echo 'UUID=YOUR_UUID_HERE /mnt/storage ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

Take ownership so you can write to it without sudo:

```bash
sudo chown -R $USER:$USER /mnt/storage
```

Verify the fstab entry mounts correctly:

```bash
sudo systemctl daemon-reload
sudo umount /mnt/storage
sudo mount -a
df -h /mnt/storage
```

If `df -h` shows `/mnt/storage` with ~1.8 TB available, the drive is set up correctly.

Result:
```bash
sahara@fedora:~$ df -h /mnt/storage
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1  1.9T  2.1M  1.8T   1% /mnt/storage
```

> **Optional: store Ollama models here.** Models are large (a 35B model is ~23 GB, a 70B model ~40 GB). To keep them off the system drive, add `Environment="OLLAMA_MODELS=/mnt/storage/ollama-models"` to the Ollama systemd override created in the next step. I chose to keep models on the system drive for now, but the option is there if space runs low.


## Step 8 — Install Ollama

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

Create a systemd override to pass the GPU variables to the Ollama service:

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo nano /etc/systemd/system/ollama.service.d/override.conf
```

Add the following content:

```ini
[Service]
Environment="HSA_OVERRIDE_GFX_VERSION=11.5.1"
Environment="HSA_ENABLE_SDMA=0"
Environment="OLLAMA_HOST=0.0.0.0"
```

> `OLLAMA_HOST=0.0.0.0` makes Ollama accessible from other devices on your local network, and is also required for Docker containers like n8n to reach it.

Reload and restart the service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### Open the firewall port (Fedora specific)

Fedora runs firewalld by default, which blocks incoming connections. To allow Docker containers and other devices on your network to reach Ollama:

```bash
sudo firewall-cmd --permanent --add-port=11434/tcp
sudo firewall-cmd --reload
```

### Pull your first models

```bash
# All-round, good starting point
ollama pull qwen3.6:35b

# For coding and writing security scripts
ollama pull qwen3-coder:30b

# Specialized OCR, extract text from images and documents
ollama pull glm-ocr

# Quick test model (small and fast)
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

During inference you should see `GPU%` at 90-97% and `VRAM%` increasing as the model loads. If GPU usage stays at 0, the `HSA_OVERRIDE_GFX_VERSION` variable was not applied correctly to the service.

> The warning `AMD GPU device(s) is/are in a low-power state` when the GPU is idle is harmless. It just means the GPU clocked down between requests, which is expected behavior.


## Step 9 — Install Docker

Docker is required for running Open WebUI and n8n as containers. Containers keep these services isolated, easy to update, and independent of your system Python or Node versions.

> Fedora ships with Podman as its native container engine, and it works too. I use Docker here for consistency with most documentation and tutorials.

### Add Docker's official repository

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo
```

### Install Docker Engine

```bash
sudo dnf install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

### Enable and start Docker

Fedora does not start Docker automatically after installation:

```bash
sudo systemctl enable --now docker
```

### Add your user to the docker group

This lets you run Docker commands without `sudo` every time.

```bash
sudo usermod -aG docker $USER
sudo reboot
```

### Verify Docker is working

```bash
docker --version
docker run hello-world
```

You should see the "Hello from Docker!" message confirming everything is working.


## Step 10 — Install Open WebUI

Open WebUI is a ChatGPT-style web interface that connects to your local Ollama instance. It runs in Docker and lets you chat with any model you have pulled, manage conversations, and switch between models mid-chat. It also includes built-in voice input powered by Whisper. Click the microphone icon in the chat bar to use it.

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

You should see your Ollama models listed in the model selector. Select one and start chatting. This is now 100% local, zero data leaving your machine.


## Step 11 — Install n8n (Self-Hosted)

n8n is a workflow automation platform. Self-hosting it means all your workflow data, credentials, and execution logs stay on your machine. Combined with the Anthropic API key (instead of claude.ai), this gives you a private AI automation stack.

### Create a dedicated directory for n8n

```bash
mkdir -p ~/n8n-data
```

### Run n8n with Docker

Note the `:z` suffix on the volume mount. Fedora uses SELinux, and without this flag the container gets permission errors accessing the data directory:

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v ~/n8n-data:/home/node/.n8n:z \
  --restart always \
  docker.n8n.io/n8nio/n8n
```

### Access n8n

Open your browser and navigate to:

```
http://localhost:5678
```

On first launch, n8n will ask you to create an owner account with an email and a password. This account is stored locally in `~/n8n-data`. 

### Connect n8n to Ollama 

n8n runs inside a Docker container, so `localhost` inside the container refers to the container itself, not your host machine where Ollama is running. You need to use your machine's local IP address instead.

Find your local IP:

```bash
hostname -I | awk '{print $1}'
```

Then when adding an **Ollama Chat Model** node to your workflow, click **Create new credential** and set the Base URL to your IP:

```
http://YOUR_LOCAL_IP:11434
```

For example:

```
http://192.168.128.253:11434
```

> `host.docker.internal` does not work on Linux. Always use your actual local IP address from `hostname -I`. If the connection test fails, verify the firewall port from Step 8 is open: `sudo firewall-cmd --list-ports` should include `11434/tcp`.

To build your first chat workflow:

1. Create a new workflow
2. Add a **When chat message received** trigger node
3. Add a **Basic LLM Chain** node and connect it to the trigger
4. Add an **Ollama Chat Model** node, connect it to the **Model** input of the Basic LLM Chain
5. In the Ollama Chat Model node, select your credential and choose a model such as `qwen3.6:35b`
6. Click **Open chat** at the bottom to test it

![n8n chat workflow with Ollama]({{ "/assets/images/n8n-ollama-first-chat.png" | relative_url }})

Your local model will respond through n8n with no API key, no usage costs, and no data leaving your machine.

### Connect n8n to Anthropic API (optional, for Claude)

For tasks where you need Claude's capabilities (better reasoning, larger context), add your Anthropic API key in n8n:

```
Settings -> Credentials -> New -> Anthropic API
```

Paste your API key from `console.anthropic.com`. When using this credential, prompts are sent to Anthropic's servers (7-day retention, never used for training) and results come back to your local n8n instance. No data is stored in claude.ai.

---

## Step 12 — Auto-start Everything on Boot

All services are configured to start automatically: Docker containers use `--restart always`, and Ollama and Docker run as systemd services. Verify:

```bash
# Check all running containers
docker ps

# Check Ollama service
systemctl status ollama
```

After a reboot, everything should come back up automatically within about 30 seconds.

---

## Step 13 — Verify the Full Stack

After rebooting, run the following checklist:

```bash
# 1. Check kernel parameters are active
cat /proc/cmdline

# 2. Check GPU recognition
rocminfo | grep gfx1151

# 3. Check Ollama is running and GPU-accelerated
ollama ps

# 4. Check Docker containers
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

## Recommended Models

Given the ~96 GB of GPU-accessible memory on this machine, you can run models that are impossible on typical consumer hardware. For detailed benchmarks of different models and backends on Strix Halo, see the community [interactive benchmark viewer](https://kyuz0.github.io/amd-strix-halo-toolboxes/).

```bash
# Best all-round, near cloud quality, 100% local
ollama pull qwen3.6:35b

# Best for coding and writing security scripts
ollama pull qwen3-coder:30b

# Deep reasoning, useful for log analysis and threat hunting
ollama pull deepseek-r1:32b

# Rivals cloud AI performance
ollama pull llama3.3:70b

# OCR and document extraction from images
ollama pull glm-ocr
```

> Start with `qwen3.6:35b`. It loads fast and performs surprisingly close to 70B models for most tasks. Use it as your daily driver and pull larger models when you need deeper reasoning.

---

## Privacy Summary

This setup achieves the following data sovereignty:

| Component                | Data location       | Internet traffic                                                |
|--------------------------|---------------------|-----------------------------------------------------------------|
| Ollama models            | Local disk          | None after initial download                                     |
| Open WebUI               | Local Docker volume | None                                                            |
| n8n workflows            | ~/n8n-data/         | None                                                            |
| Conversation history     | Local only          | None                                                            |
| Anthropic API (optional) | Local only          | Prompt and response only, 7-day log retention, never trained on |

Everything runs on your hardware. No conversation history is stored in any cloud service.

---

## Troubleshooting

**Ollama not using GPU (slow inference)**

Check that the environment variables are set correctly in the service:
```bash
sudo systemctl cat ollama | grep HSA
# Should show both HSA_OVERRIDE_GFX_VERSION and HSA_ENABLE_SDMA
```

If missing, repeat the systemd override step in Step 8 and restart the service.

**n8n cannot connect to Ollama**

First verify Ollama is reachable at your local IP:
```bash
curl http://$(hostname -I | awk '{print $1}'):11434/api/tags
```

If this times out, check the firewall:
```bash
sudo firewall-cmd --list-ports
# Should include 11434/tcp, if not repeat the firewall step
```

**n8n container not starting or permission errors**

On Fedora this is almost always SELinux. Make sure the volume mount includes the `:z` flag:
```bash
docker logs n8n
# If you see EACCES or permission denied errors, recreate the container with -v ~/n8n-data:/home/node/.n8n:z
```

**Open WebUI not connecting to Ollama**

Verify Ollama is listening:
```bash
curl http://localhost:11434/api/tags
```

If it returns JSON with your models, Ollama is working. Restart the Open WebUI container:
```bash
docker restart open-webui
```

**System freezes (mouse and keyboard stop responding)**

This is the Strix Halo stability issue covered in Part 1. Verify all three fixes are in place: firmware newer than 20251125 (`rpm -q linux-firmware`), kernel parameters active (`cat /proc/cmdline`), and never use `newgrp` after group changes. See the [community host configuration guide](https://github.com/kyuz0/amd-strix-halo-toolboxes) for the latest known-stable setup.

---

## What's Next

With the full stack running, the next project in this home lab is an **IOC enrichment pipeline** built in n8n: paste an IP address or file hash, and the workflow automatically queries VirusTotal, AbuseIPDB, and Shodan, then passes the combined results to a local Ollama model that writes a plain-language verdict. 


