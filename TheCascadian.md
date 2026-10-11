# Host Hardware Profile for AI Project Optimization

Captured live on 2026-10-10 via `inxi -Fxxxz` (no root; uptime 1h 53m).

**Legend:** OK = healthy; WARN = needs attention; UNKNOWN = not captured; † = public spec, not present in the inxi output

## Quick facts (at a glance)

| Area | Value | Status |
| --- | --- | --- |
| OS | CachyOS (Arch base), kernel 6.18.55-2-cachyos-lts | OK |
| CPU | AMD Ryzen 5 7600X, Zen 4, 6C / 12T, boost to 4.7 GHz | OK |
| RAM | 64 GiB (about 62 GiB available), no disk swap, 62 GiB zram | OK |
| GPU A (compute) | NVIDIA GTX 1660 Ti, Turing, 6 GiB VRAM†, driver 615.78.08 | OK (headless, VRAM fully free) |
| GPU B (display) | AMD Radeon 610M iGPU (RDNA 2), amdgpu, drives both monitors | OK |
| Storage | 3.25 TiB total, all SATA (no NVMe) | WARN |
| Root `/` | 208 GiB ext4, **77.8% used** (about 46 GiB free) | WARN |
| Network | Wi-Fi 6E up; 2.5GbE present but cable down; Tailscale present | WARN |
| Compute backend | **CUDA** (NVIDIA). Not XPU or ROCm. | OK |

## Executive summary

- **CPU:** AMD Ryzen 5 7600X, 6 cores / 12 threads, boost up to 4.7 GHz
- **System RAM:** 64 GiB installed; about 62 GiB available at capture
- **Primary compute GPU:** NVIDIA GTX 1660 Ti (6 GiB†). **No displays attached**, so the full VRAM is free for compute.
- **Display GPU:** AMD Radeon 610M iGPU drives both monitors, which keeps desktop VRAM usage off the NVIDIA card.
- **Critical constraint:** The 1660 Ti has **no tensor cores and no BF16 support†**. Use FP16 or quantized (INT8/INT4) inference, not BF16.
- **Secondary issue:** Root partition is 77.8% full and the whole system is on SATA storage. Model loading and caches will be bandwidth- and space-limited.
- **OS:** CachyOS (Arch base), KDE Plasma 6.7.5 on Wayland
- **Kernel:** Linux 6.18.55-2-cachyos-lts, x86_64
- **Best workload profile:** Small quantized models (7B class or less) fully on GPU; larger models (up to about 30B Q4) via CPU or hybrid CPU+GPU offload, using the large 64 GiB RAM pool.

## CPU

| Property | Value |
| --- | --- |
| Model | AMD Ryzen 5 7600X 6-Core Processor |
| Architecture | Zen 4 (rev 2) |
| Physical cores / threads | 6 / 12 (SMT enabled) |
| Clock range | 427 MHz min, 4701 MHz max, boost enabled |
| L1 / L2 / L3 cache | 384 KiB / 6 MiB / 32 MiB |
| Listed SIMD flags | AVX, AVX2, SSE4.1/4.2, SSE4a, SSSE3 |
| AVX-512 | UNKNOWN: not in inxi's basic flags; Zen 4 supports it†. Verify: `lscpu \| grep -o avx512 \| sort -u` |
| Virtualization | AMD-V / SVM |
| Package temp at capture | 40.1 °C |

Note: the per-core speed readout (all cores 4299 MHz) is a single snapshot, not a sustained clock.

Implications:

- Strong for preprocessing, orchestration, tokenization, and CPU inference of quantized models.
- Default to about 6 compute threads for CPU inference (physical cores); benchmark 12 before assuming a gain.
- Avoid thread oversubscription when CUDA and CPU work run together.

## Memory

| Property | Value |
| --- | --- |
| Installed RAM | 64 GiB |
| Estimated available | 61.9 GiB |
| Used at capture | 10.38 GiB (16.8%) |
| Disk swap | None |
| zram swap | 61.9 GiB, priority 100 (4.5 MiB used) |
| Speed / timings / DIMM layout | UNKNOWN: needs root (`dmidecode`) |

Implications:

- Room for a quantized 30B-class model fully in RAM, plus OS and tools.
- zram is compressed RAM, so it is not extra capacity. Heavy swap use will still hurt speed.
- Check EXPO/XMP is enabled in BIOS; DDR5 speed strongly affects CPU inference throughput.

## GPU

### GPU A: compute (NVIDIA)

| Property | Value |
| --- | --- |
| GPU | NVIDIA GeForce GTX 1660 Ti (TU116) |
| Architecture | Turing |
| Vendor | Micro-Star (MSI) |
| PCI ID / bus | `10de:2182` / `01:00.0` |
| Driver | proprietary `nvidia` v615.78.08 |
| VRAM | 6 GiB GDDR6† (not shown by inxi; confirm with `nvidia-smi`) |
| Tensor cores | None† |
| Precision | FP32 and FP16 yes†; BF16 and FP8 no† |
| Active outputs | None (DP-1, DP-2, DP-3, HDMI-A-1 all empty) |
| PCIe link at capture | 2.5 GT/s x16 (see note below) |
| Temperature | UNKNOWN: not exposed by inxi (`nvidia-smi` will show it) |

PCIe note: 2.5 GT/s is Gen 1 speed, but this is almost certainly an **idle power-saving downclock**, since the GPU is headless and idle. The x16 width is fine. Confirm under load: `nvidia-smi -q -d PCIE`.

### GPU B: display (AMD iGPU)

| Property | Value |
| --- | --- |
| GPU | AMD Raphael / Radeon 610M, RDNA 2 |
| PCI ID / bus | `1002:164e` / `0f:00.0` |
| Driver | `amdgpu` (kernel) |
| Role | Compositor and display output (kwin_wayland), Mesa OpenGL renderer |
| Active outputs | DP-4, HDMI-A-2 |
| Temperature | 35.0 °C (inxi reading; the Sensors summary shows 30.5 °C) |

### Graphics API summary

| API | Version | Notes |
| --- | --- | --- |
| OpenGL | 4.6 compat | Rendered by AMD radeonsi (Mesa 26.2.4) |
| Vulkan | 1.4.363 | Two devices: AMD (integrated, RADV) and NVIDIA (discrete) |
| EGL | 1.5 | nvidia, radeonsi, swrast drivers present |
| Display server | Wayland (kwin_wayland), Xwayland 24.1.14 | |
| Tools present | clinfo, vulkaninfo, nvidia-smi, nvidia-settings, wayland-info | |

Implications:

- Two GPUs are visible, so **explicitly select the NVIDIA device**. Examples: `CUDA_VISIBLE_DEVICES=0`, or `--device 1` in Vulkan tools (device 1 is NVIDIA here).
- Do not assume the Vulkan "device 0" is the compute GPU. It is the iGPU.
- Because no monitor is on the NVIDIA card, nearly all 6 GiB of VRAM can go to models.

## Displays

| Monitor | Port | Resolution | Refresh | Role |
| --- | --- | --- | --- | --- |
| CR240DM (23.8") | HDMI-A-2 | 1920x1080 | 180 Hz | Primary, left |
| Asus VS228 (21.5") | DP-4 | 1920x1080 | 60 Hz | Right |

Combined desktop area: 3840x1080.

## Storage

| Device | Model | Type | Interface | Capacity | Role |
| --- | --- | --- | --- | --- | --- |
| `/dev/sdb` | P3-1TB | SSD | SATA 6 Gb/s | 953.87 GiB | **System disk** (`/` on sdb4, `/boot` on sdb3) |
| `/dev/sdc` | TeamGroup T-FORCE 512GB | SSD | SATA 6 Gb/s | 476.94 GiB | Data (mount not shown) |
| `/dev/sda` | Seagate ST1000DM010 | HDD 7200 rpm | SATA 6 Gb/s | 931.51 GiB | Bulk or archive |
| `/dev/sdd` | WD WDS100T2B0B | SSD | USB 3.2 (10 Gb/s) | 931.51 GiB | External |
| `/dev/sde` | Kingston DT 101 G2 | USB flash | USB 2.0 | 29.08 GiB | Removable (MBR) |

Totals: 3.25 TiB, 895.13 GiB used (26.9%).

| Mount | Size | Used | Filesystem | Device | Status |
| --- | --- | --- | --- | --- | --- |
| `/` | 208.49 GiB | 162.31 GiB (77.8%) | ext4 | `/dev/sdb4` | WARN: about 46 GiB free |
| `/boot` | 4.58 GiB | 245.5 MiB (5.2%) | vfat | `/dev/sdb3` | OK |

Implications:

- No NVMe. Sequential reads cap at about 500 MB/s, so large model files take noticeably longer to load.
- About 46 GiB free on `/` is tight for model weights, venvs, and Docker images. Put model and cache directories on a larger mount.
- Use the SATA SSDs for active models; keep the HDD for archives only.
- The `/dev/sdd` USB SSD and `/dev/sde` USB flash drive are removable; do not rely on them for persistent paths.

## Network

| Interface | Hardware | State |
| --- | --- | --- |
| `wlan0` | MediaTek MT7922 802.11ax (`mt7921e`) | OK, up (active connection) |
| `enp11s0` | Realtek RTL8125 2.5GbE (`r8169`) | down (no cable or link) |
| `tailscale0` | Tailscale VPN | present |
| `docker0`, `br-d7f4dae5c934` | Docker bridges | down (no running containers) |

Bluetooth: MediaTek (`btusb`), controller `hci0`, service running, **software-blocked** (`rfkill`).

Implication: Model downloads currently go over Wi-Fi. Plug in the 2.5GbE port for faster, steadier transfers.

## Motherboard and firmware

| Property | Value |
| --- | --- |
| Manufacturer / model | Micro-Star (MSI) B650 GAMING PLUS WIFI (MS-7E26) v1.0 |
| Type | Desktop |
| Firmware | UEFI |
| BIOS vendor / version / date | American Megatrends / 1.M4 / 2025-08-07 |
| Serial / UUID | UNKNOWN: requires root |

## Operating system and toolchain

| Component | Version |
| --- | --- |
| Distro | CachyOS (base: Arch Linux) |
| Kernel | 6.18.55-2-cachyos-lts, x86_64 |
| Desktop | KDE Plasma 6.7.5 on kwin_wayland |
| Display manager | LightDM 1.33.1 |
| Init | systemd 262 |
| Audio | PipeWire 1.6.9 (with pipewire-pulse, wireplumber, pipewire-alsa) |
| Shell | fish 4.9.3 |
| Terminal | xfce4-terminal |
| Package manager | pacman (1707 packages) |
| Python | 3.14.7 |
| GCC / Clang | 16.2.1 / 23.1.1 |
| Docker | Present (interfaces exist; daemon state not captured) |

Caution: Python 3.14 is very new. Many ML wheels (PyTorch, etc.) may lag behind it. Prefer a venv pinned to a supported version (for example 3.12 or 3.13) via `uv` or `pyenv`.

## Not captured (verify before optimizing)

| Item | Why it matters | Command |
| --- | --- | --- |
| CUDA toolkit version | Wheel and build compatibility | `nvcc --version` / `nvidia-smi` |
| NVIDIA VRAM, temp, power | Exact capacity and thermals | `nvidia-smi` |
| PCIe link under load | Confirms the Gen 1 reading was idle | `nvidia-smi -q -d PCIE` |
| AVX-512 support | CPU kernel selection | `lscpu \| grep avx512` |
| RAM speed / EXPO | CPU inference throughput | `sudo dmidecode -t memory` |
| CPU governor | Sustained clocks | `cpupower frequency-info` |
| Docker daemon / GPU runtime | Containerized workloads | `docker info`, check `nvidia-container-toolkit` |

## Current resource considerations

At capture:

- About 10.4 GiB of system RAM was in use (16.8%).
- 385 processes were running; system uptime was under 2 hours.
- CPU 40.1 °C, motherboard 36.0 °C, AMD iGPU about 30 °C. All low, with no thermal concern.
- The NVIDIA GPU was idle with no display load.

## Recommended optimization targets

Another AI model or agent should tailor software toward:

1. **CUDA on the GTX 1660 Ti**, selected explicitly (two GPUs are present).
2. **FP16 or quantized INT8/INT4**, not BF16 or FP8.
3. **Models that fit about 5.5 GiB of VRAM**: 7B class at Q4, or smaller. Leave headroom for the KV cache.
4. **Hybrid CPU+GPU offload** (e.g. llama.cpp `-ngl`) for larger models, using the 64 GiB RAM.
5. **CPU-only inference** for 13B to 30B class quantized models; about 6 threads by default.
6. **Small batch sizes**, generally 1 for interactive use, unless measured otherwise.
7. **No Turing-incompatible kernels**: avoid FlashAttention variants and libraries that require Ampere or newer.
8. **Models and caches on a roomy mount**, not the nearly full `/` partition.
9. **Python 3.14 caution**: pin an ML-compatible interpreter in a venv.
10. **Treat the iGPU as display-only** unless a workload specifically targets Vulkan/ROCm on it.
11. **Wired network for large downloads**.
12. **Explicit fallback** to CPU when an operator lacks CUDA support.

## Copy-paste machine profile

```yaml
captured: 2026-10-10
source: inxi -Fxxxz (no root)

os:
  distro: CachyOS (Arch base)
  kernel: 6.18.55-2-cachyos-lts
  arch: x86_64
  desktop: KDE Plasma 6.7.5 (Wayland, kwin_wayland)
  init: systemd 262

cpu:
  model: AMD Ryzen 5 7600X
  arch: Zen 4
  cores_threads: 6/12
  max_mhz: 4701
  cache: {l1: 384KiB, l2: 6MiB, l3: 32MiB}
  avx2: true
  avx512: unknown (Zen 4 supports it; verify)

memory:
  total_gib: 64
  available_gib: 61.9
  swap: {disk: none, zram_gib: 61.9}

gpu_compute:
  name: NVIDIA GTX 1660 Ti (TU116, Turing)
  driver: nvidia 615.78.08
  pci_id: "10de:2182"
  vram_gib: 6        # spec, not from inxi
  tensor_cores: false
  bf16: false
  fp16: true
  displays_attached: 0
  pcie_at_capture: "2.5 GT/s x16 (idle downclock, verify under load)"
  cuda_toolkit: unknown

gpu_display:
  name: AMD Radeon 610M iGPU (RDNA 2)
  driver: amdgpu
  pci_id: "1002:164e"
  role: desktop compositor and monitors only

gpu_selection_hint: >
  Two GPUs visible. Vulkan device 0 = AMD iGPU, device 1 = NVIDIA.
  Select the NVIDIA GPU explicitly for compute.

storage:
  total_tib: 3.25
  used_pct: 26.9
  nvme: false
  root: {dev: /dev/sdb4, size_gib: 208.49, used_pct: 77.8, fs: ext4}
  disks:
    - {dev: sdb, model: P3-1TB, type: SATA SSD, gib: 953.87, role: system}
    - {dev: sdc, model: TeamGroup T-FORCE 512GB, type: SATA SSD, gib: 476.94}
    - {dev: sda, model: Seagate ST1000DM010, type: HDD 7200rpm, gib: 931.51}
    - {dev: sdd, model: WD WDS100T2B0B, type: USB 3.2 SSD, gib: 931.51}
    - {dev: sde, model: Kingston DT 101 G2, type: USB 2.0 flash, gib: 29.08}

network:
  active: wlan0 (MT7922 Wi-Fi 6E)
  wired: RTL8125 2.5GbE (link down)
  vpn: tailscale0

motherboard:
  model: MSI B650 GAMING PLUS WIFI (MS-7E26)
  bios: AMI 1.M4 (2025-08-07)

toolchain:
  python: 3.14.7   # very new; pin an ML-compatible version in a venv
  gcc: 16.2.1
  clang: 23.1.1
  shell: fish 4.9.3
  pkg_manager: pacman (1707 pkgs)

primary_strategy: >
  CUDA on GTX 1660 Ti, FP16 or INT4/INT8 quantization, models up to about 5.5 GiB in VRAM,
  hybrid CPU+GPU offload for larger models using 64 GiB RAM, small batches,
  caches on a roomy mount, explicit GPU selection.

flags:
  - "/ is 77.8% full; relocate models and caches"
  - "No NVMe; SATA-limited model load times"
  - "No tensor cores or BF16 on Turing"
  - "Python 3.14 may lack ML wheels"
```
