# SELinux Policy — NVIDIA GPU Access for Docker

## Why this exists

This system runs **SELinux in Enforcing mode** (Fedora default).  
When Docker tries to access NVIDIA GPU devices (`/dev/nvidia*`, `/dev/nvidiactl`, etc.), the kernel classifies them as `xserver_misc_device_t`. Without an explicit policy, SELinux denies that access and the container fails to use the GPU.

This module grants the Docker init process (`init_t`) the necessary permissions to `open`, `read`, and `write` those character devices.

---

## Files

| File | Description |
|---|---|
| `ollama-nvidia2.te` | Type Enforcement source — human-readable policy definition |
| `ollama-nvidia2.pp` | Compiled Policy Package — binary, ready to load into the kernel |

---

## What the policy allows

```
# ollama-nvidia2.te
allow init_t xserver_misc_device_t:chr_file { open read write };
```

| Subject | Object | Class | Permissions |
|---|---|---|---|
| `init_t` (Docker container init) | `xserver_misc_device_t` (NVIDIA devices) | `chr_file` (character device) | `open`, `read`, `write` |

---

## Installation

> Requires `root`. Run once per machine.

```bash
# Install the compiled policy module
sudo semodule -i selinux/ollama-nvidia2.pp

# Verify it is loaded
sudo semodule -l | grep ollama
```

Expected output:
```
ollama-nvidia2
```

---

## Recompiling from source

If you modify `ollama-nvidia2.te` or want to rebuild the `.pp` yourself:

```bash
# Requires: checkpolicy, policycoreutils
sudo dnf install checkpolicy policycoreutils-python-utils  # Fedora / RHEL

checkmodule -M -m -o ollama-nvidia2.mod selinux/ollama-nvidia2.te
semodule_package -o selinux/ollama-nvidia2.pp -m ollama-nvidia2.mod
sudo semodule -i selinux/ollama-nvidia2.pp
```

---

## Removing the policy

```bash
sudo semodule -r ollama-nvidia2
```

---

## How this module was generated

The initial policy was generated using `audit2allow`, a tool that reads SELinux AVC denial logs and produces the minimal policy required to allow the blocked operations:

```bash
# Read denials from the audit log and generate a policy
sudo ausearch -c 'docker' --raw | audit2allow -M ollama-nvidia
```

The module was then refined to include the `open` permission that was missing from the first generated version.
