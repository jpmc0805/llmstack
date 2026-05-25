# 🦙 LLMStack — Local AI Inference Stack

A fully **offline**, **GPU-accelerated**, **privacy-first** local LLM stack built with Docker.  
Run large language models entirely on your own hardware — no cloud, no telemetry, no data leaving your machine.

![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![NVIDIA](https://img.shields.io/badge/NVIDIA-GPU%20Accelerated-76B900?logo=nvidia&logoColor=white)
![SELinux](https://img.shields.io/badge/SELinux-Enforcing-CC0000?logo=redhat&logoColor=white)
![Offline](https://img.shields.io/badge/Network-Air--gapped-555555)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Network Design](#network-design)
- [Security Hardening](#security-hardening)
- [SELinux on Fedora](#selinux-on-fedora)
- [Project Structure](#project-structure)

---

## Overview

LLMStack runs two services in a private Docker network:

| Service | Image | Purpose |
|---|---|---|
| **Ollama** | `ollama/ollama:latest` | LLM inference engine — serves models via HTTP API |
| **Open-WebUI** | `ghcr.io/open-webui/open-webui:v0.9.2` | Web interface to interact with Ollama models |

Key design decisions:
- **Air-gapped network** (`internal: true`) — containers cannot reach the internet
- **Static IPs** on an isolated bridge — no port-forwarding, no NAT, accessed directly via bridge IP
- **GPU-only inference** — NVIDIA GPU is mandatory; no CPU fallback configured
- **All telemetry disabled** across every service

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  HOST MACHINE (Fedora, SELinux Enforcing)           │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  Docker bridge: ollama-net (192.168.200.0/24) │  │
│  │  internal: true  ←  NO internet access        │  │
│  │                                               │  │
│  │  ┌─────────────────┐   ┌───────────────────┐ │  │
│  │  │     ollama      │   │    open-webui     │ │  │
│  │  │  192.168.200.10 │◄──│  192.168.200.20   │ │  │
│  │  │  port 11434     │   │  port 8080        │ │  │
│  │  │  NVIDIA GPU     │   │                   │ │  │
│  │  └─────────────────┘   └───────────────────┘ │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  Access from host browser: http://192.168.200.20:8080 │
└─────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Linux host (tested on **Fedora 44**)
- **Docker** ≥ 26 with Docker Compose plugin
- **NVIDIA GPU** with proprietary drivers installed
- **NVIDIA Container Toolkit** (`nvidia-ctk`)
- SELinux in Enforcing mode (optional but recommended — see [SELinux section](#selinux-on-fedora))

### Verify your GPU is visible to Docker

```bash
docker run --rm --gpus all nvidia/cuda:12.0-base-ubuntu22.04 nvidia-smi
```

---

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/YOUR_USERNAME/llmstack.git
cd llmstack

# 2. Create your environment file
cp .env.example .env

# 3. Edit .env and set a strong secret key
#    You can generate one with: openssl rand -hex 32
nano .env

# 4. Create the external volume for Open-WebUI data (first time only)
docker volume create open-webui-data

# 5. Start the stack
cd Ollama
docker compose up -d

# 6. Pull your first model (example: llama3.2)
docker exec -it ollama ollama pull llama3.2

# 7. Open the web UI
# Navigate to: http://192.168.200.20:8080
```

---

## Configuration

All sensitive values are managed via a `.env` file that is **never committed** to the repository.

| Variable | Description | Example |
|---|---|---|
| `WEBUI_SECRET_KEY` | Session signing key for Open-WebUI | `openssl rand -hex 32` |

See [`.env.example`](.env.example) for the full template.

---

## Network Design

The stack uses a **fully isolated Docker bridge network** with static IPs:

| Component | IP | Port |
|---|---|---|
| Docker bridge gateway | `192.168.200.1` | — |
| Ollama API | `192.168.200.10` | `11434` |
| Open-WebUI | `192.168.200.20` | `8080` |

> **Why no `ports:` mapping?**  
> Docker's `ports:` directive creates NAT rules to expose a service on the host's network interfaces. With `internal: true`, Docker does not create those NAT rules, so `ports:` has no effect. Instead, services are accessed directly via the bridge's static IP from the host — no exposure to the LAN or internet.

---

## Security Hardening

| Measure | Implementation |
|---|---|
| Internet isolation | `internal: true` on Docker network — no outbound traffic from containers |
| No port exposure | Removed `ports:` mappings; access via static bridge IPs only |
| Telemetry disabled | `DO_NOT_TRACK`, `SCARF_NO_ANALYTICS`, `OLLAMA_TELEMETRY=false`, `OFFLINE_MODE=True` |
| No secret hardcoding | `WEBUI_SECRET_KEY` injected via `.env`, never in the compose file |
| GPU-only execution | `OLLAMA_GPU_OVERHEAD=0` — prevents silent CPU fallback |
| SELinux Enforcing | Custom policy module allows NVIDIA device access (see [`selinux/`](selinux/)) |

---

## SELinux on Fedora

Running Docker with NVIDIA GPUs on a system with **SELinux in Enforcing mode** requires a custom policy module. Without it, SELinux blocks the container's access to NVIDIA character devices (`/dev/nvidia*`).

See [`selinux/README.md`](selinux/README.md) for full details and installation instructions.

---

## Project Structure

```
LLMStack/
├── .env.example          # Environment variable template (copy to .env)
├── .gitignore            # Excludes .env and local artifacts
├── README.md             # This file
│
├── Ollama/
│   └── docker-compose.yml  # Ollama + Open-WebUI service definitions
│
└── selinux/
    ├── README.md           # SELinux module explanation and install guide
    ├── ollama-nvidia2.te   # Type Enforcement source policy
    └── ollama-nvidia2.pp   # Compiled policy package (ready to install)
```

---

## Contributing

Contributions, issues, and suggestions are welcome.  
Feel free to open an issue or submit a pull request.

---

## Author

**Joseph Meneses**
🎓 Student of ASIR — Systems and Database Administration
🌍 Madrid, Spain
🔗 [GitHub Profile](https://github.com/jpmc0805)

---

## License

This project is licensed under the **MIT License**.  
Feel free to use, study, modify, or adapt it for your own purposes.  
See the [`LICENSE`](LICENSE) file for details.
