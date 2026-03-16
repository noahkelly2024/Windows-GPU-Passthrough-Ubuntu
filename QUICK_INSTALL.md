# Windows VM Quick Install Guide

GPU-accelerated Windows VM on Ubuntu using KVM/QEMU with optional Looking Glass (zero-latency display).

---

## Requirements

**Hardware:**
- CPU with virtualization support (AMD SVM or Intel VT-x)
- Two GPUs: one for the host display, one to pass through to Windows
  - e.g. Intel iGPU (host) + AMD/NVIDIA dGPU (Windows)
- A monitor connected to your **motherboard** (iGPU), not your dedicated GPU
- At least 16 GB RAM and 100 GB free disk space recommended

**BIOS Settings (required before starting):**

| Setting | Value |
|---------|-------|
| CPU Virtualization | **Enabled** (AMD SVM / Intel VT-x) |
| IOMMU | **Enabled** (AMD-Vi / Intel VT-d) |
| Primary Display | **IGD** or **Auto** (use motherboard ports) |

> After changing BIOS settings, ensure your monitor is plugged into the **motherboard video output**, not the GPU card.

---

## Step 1 — Install Scripts

```bash
cd ~/Downloads
git clone https://github.com/your-repo/ubuntu-vm-scripts   # or copy the folder
cd ubuntu-vm-scripts
./ubuntu-windows-vm system-install
```

This copies all scripts to `/opt/ubuntu-vm-scripts/` and creates short commands:

| Command | What it does |
|---------|-------------|
| `windows-vm` | Windows VM install, launch, stop |
| `gpu-passthrough` | GPU passthrough setup and management |
| `looking-glass-install` | Install Looking Glass display client |
| `snapshot` | System snapshot via snapper |

---

## Step 2 — Configure GPU Passthrough

```bash
gpu-passthrough setup
```

The wizard will:
1. Detect your GPUs and IOMMU groups
2. Configure kernel parameters (`amd_iommu=on iommu=pt` or `intel_iommu=on iommu=pt`)
3. Bind your dedicated GPU to the `vfio-pci` driver at boot
4. Rebuild initramfs
5. Prompt to reboot

**Reboot when asked.** After reboot, verify the setup worked:

```bash
gpu-passthrough info status
```

Your dedicated GPU should show `Driver: vfio-pci`.

---

## Step 3 — (Optional) Install Looking Glass

Looking Glass provides a near-zero-latency display by reading the GPU framebuffer directly — no RDP compression. Skip this step if you want RDP only.

```bash
looking-glass-install
```

The wizard will:
1. Build and install the `kvmfr` kernel module (DKMS)
2. Install `looking-glass-client`
3. Ask for your display resolution to set shared memory size
4. Configure udev rules, auto-load, and permissions

A reboot may be required for the `kvmfr` module to load.

> **Note:** Connect a real monitor (or dummy plug) to your dedicated GPU. Looking Glass requires an active display output on the GPU.

---

## Step 4 — Install Windows

```bash
windows-vm install
```

**The wizard will ask:**

1. **RAM** — how much to allocate (default: half your system RAM)
2. **CPU cores** — how many to give Windows (default: quarter of total)
3. **Storage drive** — pick from a numbered list of your mounted drives; a `windows-vm/` folder is created on the chosen drive
4. **Disk size** — VM virtual disk size (64 GB+ recommended)
5. **Username / Password** — Windows login credentials
6. **TPM & Secure Boot** — recommended to leave enabled
7. **GPU passthrough** — confirm if it was configured in Step 2
8. **Looking Glass** — confirm if installed in Step 3, choose IVSHMEM size

After confirming, Windows installs automatically (10–30 minutes). The script waits patiently — don't close the terminal.

**What happens automatically during install:**
- Removes pre-installed bloat (Teams, Copilot, OneDrive, Xbox overlay, etc.)
- Installs GPU drivers via winget (AMD Adrenalin / NVIDIA / Intel Arc)
- Installs SPICE Guest Tools and Looking Glass Host (if enabled)
- Applies performance tweaks (Ultimate Performance power plan, timer resolution)
- Restores classic right-click context menu
- Reboots Windows to apply all changes

When complete, the VM shuts down and you'll see:

```
✓  Windows is installed and ready to use
   Launch Windows from the app menu or run:
      windows-vm launch
```

---

## Step 5 — Launch Windows

**Via RDP (default):**
```bash
windows-vm launch
```

**Via Looking Glass (zero-latency, requires Step 3):**
```bash
windows-vm launch --lg
```

**Keep VM running after you close the window:**
```bash
windows-vm launch --keep-alive
```

App menu shortcuts are also created — search for **Windows** or **Windows [Looking Glass]** in your application launcher.

---

## Daily Usage

```bash
windows-vm launch          # Start VM and connect (RDP)
windows-vm launch --lg     # Start VM and connect (Looking Glass)
windows-vm launch -k       # Keep VM running after disconnect
windows-vm stop            # Shut down the VM
windows-vm status          # Show VM status
```

**Looking Glass controls** (when connected):
| Key | Action |
|-----|--------|
| Scroll Lock | Release mouse/keyboard back to host |
| Scroll Lock + F | Toggle fullscreen |
| Scroll Lock + Q | Quit Looking Glass |

> Default capture key is Scroll Lock (desktops) or Right Ctrl (laptops). Change in `~/.config/looking-glass/client.ini`.

---

## Troubleshooting

**GPU still shows `amdgpu`/`nvidia` driver after reboot:**
```bash
gpu-passthrough info diagnose
```
Re-run setup if `/etc/modprobe.d/vfio.conf` is missing:
```bash
gpu-passthrough setup
```

**Docker permission denied:**
```bash
newgrp docker   # Refresh group in current shell
# or log out and back in
```

**Looking Glass: `/dev/kvmfr0` not found:**
```bash
sudo modprobe kvmfr
# If that fails, a reboot is needed (module not yet in initramfs)
```

**RDP connects but screen is black:**
- Windows may still be booting. The script waits automatically.
- Check logs: `docker logs ubuntu-windows`

**VM won't start (permission denied on entry.sh):**
The Docker image needs to be rebuilt:
```bash
docker rmi dockurr/windows:latest
windows-vm install   # Will rebuild automatically
```

---

## File Locations

| Path | Contents |
|------|----------|
| `~/.windows/` | VM disk image (or your chosen drive) |
| `~/Windows/` | Shared folder — files here are accessible inside Windows |
| `~/.config/windows/docker-compose.yml` | VM configuration |
| `~/.config/looking-glass/client.ini` | Looking Glass settings |
| `/etc/modprobe.d/vfio.conf` | GPU passthrough binding |
| `/var/log/ubuntu-gpu-passthrough/` | GPU passthrough logs |

---

## Reinstall / Remove

**Reconfigure (keep Windows data):**
```bash
windows-vm install   # Choose "Quick reinstall"
```

**Fresh install (deletes Windows):**
```bash
windows-vm install   # Choose "Fresh install"
```
> Your shared files in `~/Windows/` are always preserved.

**Remove everything:**
```bash
windows-vm remove
```
