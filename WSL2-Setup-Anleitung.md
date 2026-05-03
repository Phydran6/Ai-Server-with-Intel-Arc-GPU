# ComfyUI AI Video Server with Intel Arc GPU on WSL2

Self-hosted AI server for image-to-video and text-to-video generation using Intel Arc A770 GPU acceleration — running inside WSL2 on Windows. No dual-boot, no VirtualBox, no PCI passthrough required.

## Why WSL2?

VirtualBox does **not** support GPU passthrough on Windows hosts (the "3D Acceleration" option is just an OpenGL/DirectX bridge for the VM's display, not real GPU access — PyTorch will not see the GPU). WSL2 natively passes the host GPU through to Linux via `/dev/dxg`, and Intel Arc A-Series is officially supported by Intel's PyTorch XPU stack on WSL2.

## Features

- ✅ Text-to-Image generation
- ✅ Text-to-Video generation (LTX-Video)
- ✅ Image+Text-to-Video generation (LTX-Video)
- ✅ Intel Arc A770 GPU acceleration via WSL2
- ✅ No cloud dependencies — fully local
- ✅ Windows host stays fully usable in parallel
- ✅ Linux distribution stored on a dedicated drive (clean separation)

## Requirements

### Hardware
- Intel Arc A770 GPU (16GB VRAM)
- 32GB+ system RAM recommended
- 100GB+ free disk space (on dedicated drive recommended)

### Software
- Windows 10/11 with virtualization enabled in BIOS
- Latest Intel Arc Graphics driver (tested with `32.0.101.8737`)
- WSL2 (will be installed below)

## Quick Start

### 1. Install WSL2 (without default distro)

PowerShell as **Administrator**:

```powershell
wsl --install --no-distribution
```

Reboot when prompted.

### 2. Install Ubuntu on dedicated drive

Place the Ubuntu VHDX on a separate drive for clean separation from the Windows system drive. Adjust path to your target drive:

```powershell
mkdir H:\WSL\Ubuntu
wsl --install -d Ubuntu-24.04 --location H:\WSL\Ubuntu
```

When prompted, create user `ai` and set a password.

Verify the VHDX is on the target drive:
```powershell
dir H:\WSL\Ubuntu
```

You should see `ext4.vhdx`.

### 3. ⚠️ Critical: Disable iGPU for WSL2

If your system has both an integrated GPU (iGPU) AND the Arc dGPU, the WDDM bridge in WSL2 will crash with errors like:
```
Abort was called at 857 line in file:
./shared/source/os_interface/windows/wddm_memory_manager.cpp
```

**Workaround** (Intel-recommended):

1. Open Windows **Device Manager**
2. Expand **Display adapters**
3. Right-click the **integrated** Intel Graphics (NOT the Arc A770)
4. Select **Disable device**
5. In PowerShell: `wsl --shutdown`
6. Restart your WSL session

The iGPU can be re-enabled when not using WSL for AI workloads.

### 4. Verify GPU bridge in WSL

In Ubuntu (WSL):
```bash
ls /dev/dxg && echo "GPU bridge OK"
```

### 5. Install base dependencies

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip git wget intel-opencl-icd clinfo
```

Verify Arc detection:
```bash
clinfo -l
```

You should see `Intel(R) Graphics [0x56a0]` — that's the A770.

### 6. Add Intel GPU + oneAPI repositories

```bash
wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | sudo gpg --dearmor -o /usr/share/keyrings/intel-graphics.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu noble unified" | sudo tee /etc/apt/sources.list.d/intel-gpu.list

wget -qO - https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | sudo gpg --dearmor -o /usr/share/keyrings/intel-oneapi.gpg
echo "deb [signed-by=/usr/share/keyrings/intel-oneapi.gpg] https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/intel-oneapi.list

sudo apt update
```

### 7. Install Level Zero + oneAPI Runtime

```bash
sudo apt install -y libze1 libze-dev intel-level-zero-gpu \
  intel-oneapi-runtime-dpcpp-cpp intel-oneapi-runtime-mkl
```

### 8. Clone ComfyUI

```bash
cd ~
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
python3 -m venv venv
source venv/bin/activate
```

### 9. Install PyTorch with XPU Support

> ⚠️ Use the official PyTorch XPU build, not regular PyTorch!

```bash
pip install torch==2.7.1+xpu torchvision==0.22.1+xpu torchaudio==2.7.1+xpu \
    --index-url https://download.pytorch.org/whl/xpu
```

### 10. Verify XPU detection

```bash
python -c "import torch; print('XPU:', torch.xpu.is_available()); print('Name:', torch.xpu.get_device_name(0))"
```

Expected output:
```
XPU: True
Name: Intel(R) Graphics [0x56a0]
```

### 11. Install ComfyUI dependencies

```bash
pip install -r requirements.txt
```

### 12. Install ComfyUI Manager

```bash
cd ~/ComfyUI/custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
cd ~/ComfyUI
```

### 13. Download models

**LTX-Video 2B checkpoint** (text-to-video AND image-to-video, ~9.5GB, VAE is bundled):
```bash
cd ~/ComfyUI/models/checkpoints
wget -O ltx-video-2b-v0.9.5.safetensors "https://huggingface.co/Lightricks/LTX-Video/resolve/main/ltx-video-2b-v0.9.5.safetensors"
```

**T5-XXL Text Encoder** (~9.8GB):
```bash
mkdir -p ~/ComfyUI/models/text_encoders
cd ~/ComfyUI/models/text_encoders
wget -O t5xxl_fp16.safetensors "https://huggingface.co/comfyanonymous/flux_text_encoders/resolve/main/t5xxl_fp16.safetensors"
```

### 14. Run ComfyUI

```bash
cd ~/ComfyUI
source venv/bin/activate
PYTORCH_XPU_ALLOC_CONF=expandable_segments:True python main.py --listen 0.0.0.0 --lowvram
```

Open in Windows browser: `http://localhost:8188`

### 15. Load LTX-Video Workflow

1. In ComfyUI, click **Templates** (bottom-left sidebar)
2. Select **Video** category
3. Choose **`ltxv_text_to_video`** or **`ltxv_image_to_video`**
4. Avoid the LTX-2.3 templates — those require a 22B model that won't fit in 16GB VRAM

In the **Load Checkpoint** node, select `ltx-video-2b-v0.9.5.safetensors`.

## VRAM Tuning for Arc A770 (16GB)

The Arc A770 with `--lowvram` is tight on memory for LTX-Video. Tested working settings:

| Resolution | Length | Steps | Notes |
| --- | --- | --- | --- |
| 512x320 | 97 (4 sec) | 30 | Safe baseline |
| 576x320 | 97 (4 sec) | 40 | Near limit |
| 640x384+ | 97 | 40 | Frequently OOM |

**Generation time**: ~30 seconds for 4 sec video at 40 steps (576x320).

## Anti-OOM Tips

- Always start ComfyUI with `PYTORCH_XPU_ALLOC_CONF=expandable_segments:True` and `--lowvram`
- Resolution must be divisible by 32
- LTX-Video requires **long, descriptive prompts** (50+ words). Short prompts produce poor results — describe scene, motion, camera, lighting, and style.
- For higher resolution: consider GGUF-quantized variants (Q8 saves ~30-40% VRAM)
- For final quality: generate at supported resolution, then upscale with RealESRGAN or 4x-UltraSharp

## Optional: Systemd Service

```bash
sudo tee /etc/systemd/system/comfyui.service << 'EOF'
[Unit]
Description=ComfyUI AI Image/Video Generator
After=network.target

[Service]
Type=simple
User=ai
WorkingDirectory=/home/ai/ComfyUI
Environment="PYTORCH_XPU_ALLOC_CONF=expandable_segments:True"
ExecStart=/home/ai/ComfyUI/venv/bin/python main.py --listen 0.0.0.0 --lowvram
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable comfyui
sudo systemctl start comfyui
```

## Troubleshooting

### `Abort was called at ... wddm_memory_manager.cpp`
Both iGPU and dGPU are exposed to WSL. Disable the iGPU in Windows Device Manager (see Step 3).

### `XPU available: False`
- Verify Level Zero is installed: `dpkg -l | grep intel-level-zero-gpu`
- Verify Windows Arc driver is up to date
- Restart WSL: `wsl --shutdown` in PowerShell, then reopen Ubuntu

### `torch.OutOfMemoryError: XPU out of memory`
- Restart ComfyUI with `--lowvram` flag
- Set `PYTORCH_XPU_ALLOC_CONF=expandable_segments:True` before starting
- Reduce resolution (must be divisible by 32) and length
- Reduce steps

### ComfyUI not reachable from Windows browser
- Make sure ComfyUI was started with `--listen 0.0.0.0`
- Try `http://127.0.0.1:8188` instead of `localhost:8188`
- Check Windows Firewall isn't blocking WSL connections

## Notes on iGPU

If your system has no integrated GPU (e.g. AMD CPU + Arc dGPU only), Step 3 can be skipped — the WDDM crash only occurs with multiple GPUs exposed to WSL2.

## Resources

- [ComfyUI](https://github.com/comfyanonymous/ComfyUI)
- [ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo)
- [PyTorch XPU Setup Guide](https://discuss.pytorch.org/t/solved-pytorch-2-7-1-xpu-intel-arc-graphics-complete-setup-guide-linux/220821)
- [Intel Arc on WSL2 — Intel Documentation](https://intel.github.io/intel-extension-for-pytorch/xpu/latest/tutorials/installation.html)
- [LTX-Video on Hugging Face](https://huggingface.co/Lightricks/LTX-Video)

## License

This documentation is provided as-is. ComfyUI, LTX-Video and related tools have their own licenses.
