# Ai-Server-with-Intel-Arc-GPU

# ComfyUI AI Video Server with Intel Arc GPU

Self-hosted AI server for image-to-video and text-to-video generation using Intel Arc A770 GPU acceleration.

## Features

- ✅ Text-to-Image generation (seconds)
- ✅ Text-to-Video generation (AnimateDiff)
- ✅ Image+Text-to-Video generation
- ✅ Intel Arc A770 GPU acceleration (16GB VRAM)
- ✅ No cloud dependencies - fully local
- ✅ No content restrictions

## Requirements

### Hardware
- Intel Arc A770 GPU (or other Intel Arc GPU)
- 16GB+ RAM recommended
- 50GB+ free disk space

### Software
- Ubuntu 24.04 (or compatible Linux)
- Python 3.12+

## Quick Start

### 1. Clone ComfyUI

```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
python3 -m venv venv
source venv/bin/activate
```

### 2. Setup Intel Arc GPU

```bash
# Add user to GPU groups
sudo usermod -aG render $USER
sudo usermod -aG video $USER

# Intel GPU Repository
wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | sudo gpg --dearmor -o /usr/share/keyrings/intel-graphics.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu noble unified" | sudo tee /etc/apt/sources.list.d/intel-gpu.list
sudo apt update

# Install drivers
sudo apt install -y libze1 libze-dev intel-opencl-icd intel-level-zero-gpu

# Intel oneAPI Runtime
wget -qO - https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | sudo gpg --dearmor -o /usr/share/keyrings/intel-oneapi.gpg
echo "deb [signed-by=/usr/share/keyrings/intel-oneapi.gpg] https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/intel-oneapi.list
sudo apt update
sudo apt install -y intel-oneapi-runtime-dpcpp-cpp intel-oneapi-runtime-mkl intel-pti-0.10
```

### 3. Install PyTorch with XPU Support

> ⚠️ **Important:** Use the official PyTorch XPU build, not regular PyTorch!

```bash
pip install torch==2.7.1+xpu torchvision==0.22.1+xpu torchaudio==2.7.1+xpu \
    intel-cmplr-lib-rt intel-cmplr-lib-ur intel-cmplr-lic-rt intel-sycl-rt \
    pytorch-triton-xpu tcmlib umf intel-pti \
    --index-url https://download.pytorch.org/whl/xpu \
    --extra-index-url https://pypi.org/simple
```

### 4. Verify GPU Detection

```bash
python -c "import torch; print(f'XPU available: {torch.xpu.is_available()}'); print(f'Device: {torch.xpu.get_device_name(0)}')"
```

Expected output:
```
XPU available: True
Device: Intel(R) Arc(TM) A770 Graphics
```

### 5. Install ComfyUI Dependencies

```bash
pip install -r requirements.txt
```

### 6. Install ComfyUI Manager

```bash
cd custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
cd ..
```

### 7. Download Models

```bash
# Stable Diffusion Checkpoint
cd models/checkpoints
wget -O realisticVision_v51.safetensors "https://civitai.com/api/download/models/130072"

# AnimateDiff Motion Module
mkdir -p ../animatediff_models
cd ../animatediff_models
wget https://huggingface.co/guoyww/animatediff/resolve/main/mm_sd_v15_v2.ckpt
```

### 8. Run ComfyUI

```bash
cd ~/ComfyUI
source venv/bin/activate
python main.py --listen 0.0.0.0
```

Open browser: `http://localhost:8188`

### 9. Install Video Nodes (in WebUI)

1. Click **Manager** → **Custom Nodes Manager**
2. Search and install:
   - `ComfyUI-VideoHelperSuite`
   - `ComfyUI-AnimateDiff-Evolved`
3. Restart ComfyUI

## Systemd Service (Optional)

```bash
sudo tee /etc/systemd/system/comfyui.service << 'EOF'
[Unit]
Description=ComfyUI AI Image/Video Generator
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=/home/$USER/ComfyUI
ExecStart=/home/$USER/ComfyUI/venv/bin/python main.py --listen 0.0.0.0
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable comfyui
sudo systemctl start comfyui
```

## VRAM Recommendations

| Resolution | Max Frames | Estimated VRAM |
|------------|------------|----------------|
| 384x384 | 8-12 | ~8-10GB |
| 512x512 | 4-8 | ~10-14GB |

For out-of-memory errors:
- Restart ComfyUI to clear VRAM
- Reduce frame count
- Use `--lowvram` flag

## Performance

| Task | Intel Arc A770 |
|------|----------------|
| Text-to-Image 512x512 | ~5-15 sec |
| Text-to-Video 4 frames | ~1-3 min |
| Image-to-Video 8 frames | ~2-5 min |

## Troubleshooting

### GPU not detected

```bash
# Check if user is in correct groups
groups
# Should include: render video

# If not:
sudo usermod -aG render $USER
sudo usermod -aG video $USER
# Then log out and back in
```

### XPU not available

```bash
# Check Level Zero driver
dpkg -l | grep intel-level-zero-gpu

# Install if missing
sudo apt install -y intel-level-zero-gpu
```

### Import errors

Reinstall PyTorch with correct XPU version (see step 3).

## Project Structure

```
ComfyUI/
├── models/
│   ├── checkpoints/
│   │   └── realisticVision_v51.safetensors
│   └── animatediff_models/
│       └── mm_sd_v15_v2.ckpt
├── custom_nodes/
│   ├── ComfyUI-Manager/
│   ├── ComfyUI-VideoHelperSuite/
│   └── ComfyUI-AnimateDiff-Evolved/
├── output/
├── input/
└── venv/
```

## Resources

- [ComfyUI](https://github.com/comfyanonymous/ComfyUI)
- [ComfyUI-AnimateDiff-Evolved](https://github.com/Kosinkadink/ComfyUI-AnimateDiff-Evolved)
- [PyTorch XPU Setup Guide](https://discuss.pytorch.org/t/solved-pytorch-2-7-1-xpu-intel-arc-graphics-complete-setup-guide-linux/220821)
- [Intel Arc GPU Support in PyTorch](https://pytorch.org/blog/intel-gpu-support-pytorch-2-5/)
- [CivitAI Models](https://civitai.com)

## License

This documentation is provided as-is. ComfyUI and related tools have their own licenses.

## Contributing

Feel free to open issues or PRs for improvements.
