# FreeBSD 15.0 Linuxulator + CUDA setup

- FreeBSD 15.0-RELEASE 上の Linux 互換層（Linuxulator）を利用し、RTX 4060 Ti 等の NVIDIA GPU で CUDA および llama.cpp を動作させるための手順。
  これは試行錯誤の結果なので、不十分な点や間違いがあるかもしれませんが、気づいたときに記事を修正するかもしれません。
- Steps to run CUDA and llama.cpp on NVIDIA GPUs such as the RTX 4060 Ti using the Linux compatibility layer (Linuxulator) on FreeBSD 15.0-RELEASE.
  Since this is the result of trial and error, there may be deficiencies or mistakes, but I will correct the article whenever I notice them.

## 1. Rocky Linux 9 base system

- FreeBSD側でコンテナイメージを展開し、Linux環境のルートを作成する。
 pkgやportsでインストールするとdnfなどが無いので自前で展開。
- Expand the container image on the FreeBSD side and create the root of the Linux environment.
  Since tools like dnf are not available when installing with pkg or ports, you have to set it up yourself.

```bash
# download image
fetch https://dl.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-Container-Base.latest.x86_64.tar.xz -o rocky-base.oci.tar.xz
mkdir -p rocky-rootfs
tar xf rocky-base.oci.tar.xz -C rocky-rootfs

# extract files
find rocky-rootfs/blobs/sha256 -type f -exec tar xvzf {} -C rocky-rootfs \; 2>/dev/null

# move to /compat/linux
mv rocky-rootfs /compat/linux
```

## 2. CUDA install for linux emulation

- `chroot` で Linux 環境に入り、CUDA ツールキットをセットアップする。
  linux.koのロードやfstabに"tmpfs /compat/linux/tmp tmpfs rw,mode=1777 0 0"は忘れずに。
- Enter the Linux environment using `chroot` and set up the CUDA toolkit. Don't forget to load linux.ko and add `tmpfs /compat/linux/tmp tmpfs rw,mode=1777 0 0` to `/etc/fstab`.

```bash
chroot /compat/linux /bin/bash

# add repositories
dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo

# CUDA Toolkit 12.4 install
# Be careful, because if you install with dnf, it won't install since systemd is not available.
wget https://developer.download.nvidia.com/compute/cuda/12.4.0/local_installers/cuda_12.4.0_550.54.14_linux.run
chmod +x cuda_12.4.0_550.54.14_linux.run
./cuda_12.4.0_550.54.14_linux.run --toolkit --silent --override

# PATH
export PATH=/usr/local/cuda-12.4/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:$LD_LIBRARY_PATH
```

## 3. FreeBSD host NVIDIA driver

```bash
# nvidia driver install from FreeBSD host.
pkg install nvidia-driver linux-nvidia-driver
kldload nvidia
```

## 4. CUDA dummy-uvm

- On Linuxulator, NVIDIA Unified Memory (UVM) is not implemented, causing an initialization error (Error 304). To work around this, use `dummy-uvm.so`.
- Download `uvm_ioctl_override.c` https://gist.githubusercontent.com/shkhln/40ef290463e78fb2b0000c60f4ad797e/raw/0e1fd8e8ea52b7445c3d33f5e5975efd20388dcb/uvm_ioctl_override.c


```
# Compile on linux
chroot /compat/linux /bin/bash
dnf install cmake git
dnf install -y cuda-12-1 --skip-broken
gcc -m64 -std=c99 -Wall -ldl -fPIC -shared -fno-lto -o dummy-uvm.so uvm_ioctl_override.c
```

### PyTorch test

```python
# test_tensor.py
import torch
device = torch.device("cuda")
a = torch.ones(3, 3).to(device)
print(f"Calculation Success! \n{a + a}")
```

**exec:**

```bash
python3 -m venv .venv
source .venv/bin/activates
pip install torch
LD_PRELOAD="dummy-uvm.so" python3 test_tensor.py
```

## 5. llama.cpp build

- CPU命令セットの不整合（AVX/FMA）を回避するため、最適化フラグを調整してビルドする。
- Adjust the optimization flags and build to avoid CPU instruction set mismatches (AVX/FMA).

```bash
dnf install cmake gcc-c++ git
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp && mkdir build && cd build

# cmake
# AVX/FMA should be adjusted according to the execution environment (turned off this time for safety)
LD_PRELOAD="dummy-uvm.so" cmake .. \
    -DGGML_CUDA=ON \
    -DGGML_AVX=OFF \
    -DGGML_AVX2=OFF \
    -DGGML_FMA=OFF \
    -DGGML_F16C=OFF \
    -DGGML_NATIVE=OFF \
    -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.4/bin/nvcc

# build
cmake --build . --config Release -j$(nproc)
```

## 6. exec

- サーバー起動時にも `LD_PRELOAD` を付与する。
- Attach `LD_PRELOAD` when starting the server as well.

```bash
LD_PRELOAD="dummy-uvm.so" ./bin/llama-server \
    -m /path/to/model.gguf \
    -ngl 99 \
    --host 0.0.0.0 --port 11434
```

one liner
 
```bash
chroot /compat/linux /usr/bin/env LD_PRELOAD="/path/to/dummy-uvm.so" /path/to/llama-server -m /path/to/your_model.gguf -n 128 -ngl 99 --host Your_Bind_Address --port Your_Port
```

## 7. images

### `nvidia-msi` with FreeBSD and LinuxEmulation

<img width="1297" height="869" alt="20260218-FreeBSD15-CUDA" src="https://github.com/user-attachments/assets/a11fda4d-039f-4ae3-bf13-e21fbd12c6ed" />

### llama-server found "CUDA devices"

<img width="1297" height="781" alt="20260218-FreeBSD15-CUDA3" src="https://github.com/user-attachments/assets/5ed3be4b-81eb-491b-aa74-e84dae56cbd9" />

### llama-server generate tokens.

<img width="1297" height="781" alt="20260218-FreeBSD15-CUDA2" src="https://github.com/user-attachments/assets/996032c2-da41-4106-93d0-0fbda3c65f03" />




#
