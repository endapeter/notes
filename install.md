Below is the updated `install.md` content. Replace your file with this.

---

# Docker + SSH compute-worker setup, CUDA-enabled

Assumes:

- WSL Ubuntu dev laptop.
- Linux Mint spare laptop.
- Same CPU architecture: `x86_64` / `amd64`.
- Optional NVIDIA GPU on the spare laptop.
- Docker image contains runtime/packages.
- Code/data are mounted from disk.
- CUDA driver is installed on the host.
- CUDA runtime/libraries are inside the Docker image.

---

## 1. Variables

Edit these:

```bash
MINT_USER="peter"
SPARE_LAN="100.65.185.253"
IMAGE="myproject:latest"
PROJECT_DIR="~/projects/myproject"
CUDA_IMAGE="nvidia/cuda:12.4.1-base-ubuntu22.04"
GPU_FLAGS="--gpus all"
```

Notes:

```text
SPARE_LAN   Can be LAN IP or Tailscale IP. Prefer Tailscale.
GPU_FLAGS   Use "--gpus all" if the spare has an NVIDIA GPU.
            Use "" if no GPU.
CUDA_IMAGE  Test image used to verify Docker GPU support.
```

---

# One-time setup on spare Linux Mint laptop

## 2. Install required packages

```bash
sudo apt update
sudo apt install -y \
  curl \
  ca-certificates \
  gnupg \
  git \
  rsync \
  tmux \
  openssh-server \
  ufw \
  docker.io \
  zstd

sudo systemctl enable --now ssh docker
sudo ufw allow OpenSSH
sudo ufw --force enable
sudo usermod -aG docker "$USER"
```

Log out and back in so the Docker group applies.

Verify:

```bash
docker --version
systemctl status docker --no-pager
```

---

## 3. Prevent sleep/suspend

```bash
sudo mkdir -p /etc/systemd/logind.conf.d

cat <<'EOF' | sudo tee /etc/systemd/logind.conf.d/no-sleep.conf
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
EOF

sudo systemctl restart systemd-logind
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

---

## 4. Optional but recommended: add swap

Useful for avoiding host RAM OOM, but may be slow.

```bash
sudo fallocate -l 16G /swapfile || sudo dd if=/dev/zero of=/swapfile bs=1M count=16384 status=progress
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

grep -q '^/swapfile' /etc/fstab || echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

# CUDA host setup on Linux Mint

## 5. Install NVIDIA driver

CUDA containers require the NVIDIA driver on the host. The container does not need the full kernel driver.

```bash
sudo apt update
sudo apt install -y ubuntu-drivers-common linux-headers-$(uname -r)
```

List available drivers:

```bash
ubuntu-drivers devices
```

Install the recommended driver:

```bash
sudo ubuntu-drivers install
```

Reboot:

```bash
sudo reboot
```

After reboot, verify:

```bash
nvidia-smi
```

You should see the GPU, driver version, and maximum supported CUDA version.

Example:

```text
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.xx     Driver Version: 550.xx     CUDA Version: 12.4     |
+---------------------------------------------------------------------------------------+
```

Important:

```text
The CUDA Version shown by nvidia-smi is the maximum CUDA version supported by the driver.
Your container CUDA image must be less than or equal to this version.
```

If `ubuntu-drivers` does not find a good driver:

```bash
apt-cache search '^nvidia-driver-'
sudo apt install nvidia-driver-550
sudo reboot
```

Replace `550` with the appropriate recommended version.

Secure Boot note:

```text
If Secure Boot is enabled, Ubuntu/Mint may ask you to create a MOK password.
Reboot, enroll the MOK key, then check nvidia-smi again.
```

If the laptop has hybrid graphics and `nvidia-smi` fails:

```bash
prime-select query
sudo prime-select nvidia
sudo systemctl restart display-manager
```

---

## 6. Install NVIDIA Container Toolkit

This allows Docker to expose the NVIDIA GPU to containers.

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor --yes -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
```

Configure Docker:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Verify Docker runtime:

```bash
docker info | grep -i runtime
```

You should see `nvidia` listed.

---

## 7. Test Docker CUDA access

```bash
docker run --rm --gpus all "$CUDA_IMAGE" nvidia-smi
```

Expected result: `nvidia-smi` output from inside the container.

If this fails:

```bash
nvidia-smi
sudo systemctl status docker --no-pager
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
journalctl -u docker --no-pager | tail -50
```

---

# Remote access from any network: Tailscale

## 8. Install Tailscale on spare laptop

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4
```

Install Tailscale on every device you will SSH from and log into the same Tailnet.

Then SSH using the Tailscale IP or MagicDNS name:

```bash
ssh youruser@spare.tailnet-name.ts.net
```

If you cannot install Tailscale on a device, use router port forwarding only as a last resort.

---

# One-time setup on WSL dev laptop

## 9. Install SSH client

```bash
sudo apt update
sudo apt install -y openssh-client rsync
```

---

## 10. SSH key access to spare laptop

On LAN:

```bash
ssh-keygen -t ed25519
ssh-copy-id "$MINT_USER@$SPARE_LAN"
```

Or after Tailscale:

```bash
ssh-copy-id "$MINT_USER@spare.tailnet-name.ts.net"
```

Add `~/.ssh/config`:

```ssh-config
Host spare
    HostName spare.tailnet-name.ts.net
    User youruser
```

Test:

```bash
ssh spare
```

---

# Docker image transfer, export, and import

## 11. Preferred: build from Dockerfile on spare

Best if you have a Dockerfile:

```bash
ssh spare
cd ~/projects/myproject
docker build -t myproject:latest .
```

This avoids transferring large images and ensures native compilation on the spare laptop.

---

## 12. If image exists on dev laptop: transfer with `docker save`

From WSL:

```bash
docker save "$IMAGE" | gzip -1 | ssh spare 'gunzip | docker load'
```

Faster if `zstd` is installed on both machines:

```bash
docker save "$IMAGE" | zstd -T0 | ssh spare 'zstd -d | docker load'
```

Verify on spare:

```bash
ssh spare 'docker images'
```

---

## 13. Export image to a file, then move it

Useful if the network transfer is unreliable.

On dev laptop:

```bash
docker save "$IMAGE" | zstd -T0 > myproject-image.tar.zst
```

Transfer:

```bash
rsync -P myproject-image.tar.zst spare:~/docker-images/
```

On spare:

```bash
ssh spare 'mkdir -p ~/docker-images'
ssh spare 'zstd -d < ~/docker-images/myproject-image.tar.zst | docker load'
```

---

## 14. If you manually installed packages inside a running container

Commit the container to an image first:

```bash
docker ps -a
docker commit <container_id_or_name> "$IMAGE"
```

Then transfer:

```bash
docker save "$IMAGE" | zstd -T0 | ssh spare 'zstd -d | docker load'
```

Important:

```text
docker commit saves filesystem changes as a new image.
It does not include data stored in Docker named volumes.
Sync volume data separately if needed.
```

---

## 15. Alternative: use a container registry

On dev laptop:

```bash
docker tag "$IMAGE" ghcr.io/yourname/myproject:latest
docker push ghcr.io/yourname/myproject:latest
```

On spare laptop:

```bash
docker pull ghcr.io/yourname/myproject:latest
```

---

## 16. `docker export` fallback: export container filesystem

Use this only if you cannot get the image.

`docker export` flattens the container filesystem. It loses:

```text
image layers
CMD
ENTRYPOINT
ENV
WORKDIR
history
```

Export and import:

```bash
docker export <container_id_or_name> | ssh spare 'docker import - myproject:imported'
```

Or export to file:

```bash
docker export <container_id_or_name> > container-fs.tar
rsync -P container-fs.tar spare:~/docker-images/
ssh spare 'docker import - myproject:imported' < container-fs.tar
```

Because metadata is lost, run the image with an explicit command:

```bash
ssh spare '
  docker run -d --name job1 \
    --gpus all \
    -v ~/projects/myproject:/project \
    -w /project \
    myproject:imported \
    ./run_job.sh
'
```

Prefer `docker commit` + `docker save` over `docker export`.

---

## 17. Backup and restore Docker named volumes

If your container uses named volumes, back them up separately.

Backup:

```bash
docker run --rm \
  -v myvolume:/data \
  -v "$HOME/backups:/backup" \
  alpine tar czf /backup/myvolume.tar.gz -C /data .
```

Restore:

```bash
docker run --rm \
  -v myvolume:/data \
  -v "$HOME/backups:/backup" \
  alpine tar xzf /backup/myvolume.tar.gz -C /data
```

If your project uses bind-mounted directories, `rsync` is sufficient.

---

# CUDA Docker image setup

## 18. Choose the correct CUDA base image

Check the maximum CUDA version supported by the host driver:

```bash
nvidia-smi
```

Example:

```text
CUDA Version: 12.4
```

Use a container CUDA image less than or equal to that version.

Common NVIDIA base images:

```dockerfile
# Minimal CUDA runtime, mostly for testing
FROM nvidia/cuda:12.4.1-base-ubuntu22.04

# Recommended for running CUDA applications
FROM nvidia/cuda:12.4.1-runtime-ubuntu22.04

# Required if you need nvcc or compile CUDA code
FROM nvidia/cuda:12.4.1-devel-ubuntu22.04
```

For Python ML workloads, often use framework images instead:

```dockerfile
FROM pytorch/pytorch:2.3.1-cuda12.1-cudnn8-runtime
```

or:

```dockerfile
FROM tensorflow/tensorflow:2.16.1-gpu
```

---

## 19. Example CUDA Dockerfile

```dockerfile
FROM nvidia/cuda:12.4.1-runtime-ubuntu22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    git \
    rsync \
    curl \
    ca-certificates \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /project

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["./run_job.sh"]
```

Build:

```bash
docker build -t myproject:latest .
```

If you need to compile CUDA code, use:

```dockerfile
FROM nvidia/cuda:12.4.1-devel-ubuntu22.04
```

---

# Project/code/data sync

## 20. Create remote project directory

```bash
ssh spare 'mkdir -p ~/projects/myproject'
```

---

## 21. Sync project from WSL

From your local project directory:

```bash
rsync -a --delete --info=progress2 \
  --exclude='.git' \
  --exclude='.venv' \
  --exclude='__pycache__' \
  --exclude='node_modules' \
  --exclude='*.log' \
  --exclude='out' \
  ./ spare:~/projects/myproject/
```

Git is optional.

```text
Use Git if you want committed reproducible snapshots.
Use rsync if you want current uncommitted files.
```

Do not sync unnecessary large data.

---

# Job runner on spare laptop

## 22. Create `~/bin/run-job.sh` on spare laptop

```bash
ssh spare 'mkdir -p ~/bin'
ssh spare 'nano ~/bin/run-job.sh'
```

Paste:

```bash
#!/usr/bin/env bash
set -euo pipefail

IMAGE="${IMAGE:?Set IMAGE}"
NAME="${NAME:-job1}"
PROJECT_DIR="${PROJECT_DIR:-$HOME/projects/myproject}"
CMD="${CMD:-./run_job.sh}"
RESERVE_MB="${RESERVE_MB:-2048}"
CPUS="${CPUS:-$(nproc)}"
GPU_FLAGS="${GPU_FLAGS:-}"

TOTAL_MB=$(awk '/MemTotal/{printf "%d", $2/1024}' /proc/meminfo)
MEM_MB=$((TOTAL_MB - RESERVE_MB))

if [ "$MEM_MB" -lt 512 ]; then
  MEM_MB=512
fi

# --memory-swap is total memory + swap.
# This allows RESERVE_MB worth of swap beyond the RAM limit.
SWAP_MB=$((MEM_MB + RESERVE_MB))

docker rm -f "$NAME" >/dev/null 2>&1 || true

docker run -d \
  --name "$NAME" \
  --init \
  ${GPU_FLAGS} \
  -e NVIDIA_VISIBLE_DEVICES=all \
  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility \
  --memory="${MEM_MB}m" \
  --memory-swap="${SWAP_MB}m" \
  --cpus="$CPUS" \
  -e JOB_WORKERS="$CPUS" \
  -e JOB_MEMORY_LIMIT_MB="$MEM_MB" \
  -v "$PROJECT_DIR:/project" \
  -w /project \
  "$IMAGE" \
  sh -c "$CMD"
```

Make executable:

```bash
ssh spare 'chmod +x ~/bin/run-job.sh'
```

---

# Submitting jobs

## 23. Sync code, then start job

From WSL:

```bash
rsync -a --delete --info=progress2 \
  --exclude='.git' \
  --exclude='.venv' \
  --exclude='__pycache__' \
  --exclude='node_modules' \
  --exclude='*.log' \
  --exclude='out' \
  ./ spare:~/projects/myproject/
```

Start default job without GPU:

```bash
ssh spare 'IMAGE=myproject:latest NAME=job1 GPU_FLAGS="" "$HOME/bin/run-job.sh"'
```

Start default job with GPU:

```bash
ssh spare 'IMAGE=myproject:latest NAME=job1 GPU_FLAGS="--gpus all" "$HOME/bin/run-job.sh"'
```

Start a custom command with GPU:

```bash
ssh spare 'IMAGE=myproject:latest NAME=job1 GPU_FLAGS="--gpus all" CMD="./run_job.sh --batch-size 256 --workers 4" "$HOME/bin/run-job.sh"'
```

To use a specific GPU:

```bash
ssh spare 'IMAGE=myproject:latest NAME=job1 GPU_FLAGS="--gpus device=0" "$HOME/bin/run-job.sh"'
```

---

# Monitoring

## 24. Container logs

```bash
ssh spare 'docker logs -f job1'
```

---

## 25. Container CPU/RAM usage

```bash
ssh spare 'docker stats --no-stream job1'
```

---

## 26. GPU usage

```bash
ssh spare 'nvidia-smi'
```

Continuous:

```bash
ssh spare 'watch -n 1 nvidia-smi'
```

CSV output:

```bash
ssh spare 'nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total --format=csv'
```

---

## 27. Check exit/OOM state

```bash
ssh spare 'docker inspect --format "OOM={{.State.OOMKilled}} Status={{.State.Status}} Exit={{.State.ExitCode}}" job1'
```

Possible results:

```text
OOM=true     container was killed by memory limit
OOM=false    not killed by container OOM
Status=exited
Exit=137     commonly OOM/killed signal 9
```

Host-level OOM:

```bash
ssh spare 'sudo journalctl -k | grep -i oom'
```

---

# Retrieving results

## 28. Pull outputs back to WSL

```bash
rsync -a --info=progress2 spare:~/projects/myproject/out/ ./out/
```

Or logs:

```bash
rsync -a --info=progress2 spare:~/projects/myproject/*.log ./logs/
```

---

# OOM and GPU memory rules

Docker does not magically prevent OOM. It only confines the limit.

Important facts:

```text
--memory=12g          hard host RAM/cgroup limit
--memory-swap=16g     total RAM + swap limit
--memory-swap=12g     effectively no swap
--cpus=4              CPU limit
--gpus all            expose all NVIDIA GPUs
```

Docker memory flags limit host RAM, not GPU VRAM.

GPU VRAM OOM appears in application logs, for example:

```text
CUDA out of memory
```

To avoid host RAM OOM:

1. Set `--memory` below total host RAM.
2. Reserve 1–4 GB for Linux Mint.
3. Make the job read `JOB_MEMORY_LIMIT_MB`.
4. Reduce batch size, workers, threads, or concurrency.
5. Stream/chunk data instead of loading everything.
6. Write checkpoints/resume state.
7. Add swap if slow-but-successful is acceptable.
8. Run one heavy job at a time.

To avoid GPU VRAM OOM:

1. Reduce batch size.
2. Reduce model size or sequence length.
3. Use mixed precision if supported.
4. Use gradient checkpointing if training.
5. Limit parallel workers.
6. Move inactive tensors off GPU.
7. Monitor with `nvidia-smi`.

Docker cannot generally limit consumer NVIDIA GPU VRAM. Your application must manage VRAM.

Your application should do something like:

```text
if JOB_MEMORY_LIMIT_MB is set:
    reduce batch size / concurrency
else:
    detect available memory
```

Or read cgroup memory limit inside the container:

```bash
cat /sys/fs/cgroup/memory.max
```

on cgroup v2 systems.

---

# Recommended workflow

```bash
# 1. Change code locally in WSL.

# 2. Sync to spare.
rsync -a --delete \
  --exclude='.git' \
  --exclude='.venv' \
  --exclude='__pycache__' \
  --exclude='node_modules' \
  --exclude='*.log' \
  --exclude='out' \
  ./ spare:~/projects/myproject/

# 3. Start GPU job.
ssh spare 'IMAGE=myproject:latest NAME=job1 GPU_FLAGS="--gpus all" "$HOME/bin/run-job.sh"'

# 4. Monitor.
ssh spare 'docker logs -f job1'
ssh spare 'docker stats --no-stream job1'
ssh spare 'nvidia-smi'

# 5. Retrieve results.
rsync -a spare:~/projects/myproject/out/ ./out/
```

---

# Minimal answer

Install NVIDIA driver and NVIDIA Container Toolkit on Linux Mint:

```bash
sudo ubuntu-drivers install
sudo reboot
```

After reboot:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor --yes -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Test:

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

Transfer the Docker image with:

```bash
docker save myproject:latest | zstd -T0 | ssh spare 'zstd -d | docker load'
```

Run it with GPU and conservative limits:

```bash
docker run -d --name job1 \
  --gpus all \
  --memory=10g \
  --memory-swap=14g \
  --cpus=4 \
  -e JOB_WORKERS=4 \
  -e JOB_MEMORY_LIMIT_MB=10240 \
  -v ~/projects/myproject:/project \
  -w /project \
  myproject:latest \
  ./run_job.sh
```

Export a modified container as an image:

```bash
docker commit <container_id_or_name> myproject:latest
docker save myproject:latest | zstd -T0 | ssh spare 'zstd -d | docker load'
```

Use `docker export` only as a fallback:

```bash
docker export <container_id_or_name> | ssh spare 'docker import - myproject:imported'
```