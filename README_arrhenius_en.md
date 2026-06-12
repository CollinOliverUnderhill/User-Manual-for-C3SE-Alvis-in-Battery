# Arrhenius Environment Setup and Usage Guide

This guide is intended for group members who need to access NAISS Arrhenius from Windows or macOS. It covers login, file transfer, first checks, software modules, Python environments, containers, and Slurm job submission.

The most important Arrhenius-specific point is the GPU partition: it is based on NVIDIA GH200 Grace Hopper Superchips. The CPU side of those GPU nodes is ARM-based, not traditional x86. Any program, Python package, compiled extension, or container intended for the `gpu` partition must therefore be compatible with `aarch64` / ARM.

## 0. Overview

| Purpose | Recommended method |
| --- | --- |
| Command-line login | SSH |
| Windows local setup | Windows Terminal / PowerShell + WSL Ubuntu |
| macOS local setup | Terminal |
| File transfer | `rsync`, `scp`, `sftp`, FileZilla |
| Remote desktop | ThinLinc Client |
| Account, project, and allocation management | SUPR |
| Batch jobs | Slurm: `sbatch`, `salloc`, `squeue`, `sinfo` |

Common host names:

```text
SSH:       login.hpc.arrhenius.naiss.se
Fallback:  arrhenius1.hpc.arrhenius.naiss.se
Fallback:  arrhenius3.hpc.arrhenius.naiss.se
ThinLinc:  tl.hpc.arrhenius.naiss.se
```

> `tl.hpc.arrhenius.naiss.se` is a ThinLinc Client server address. It is not necessarily a normal browser dashboard.

## 1. Prerequisites

Before configuring your local machine, make sure you have completed these steps in SUPR:

1. Request an Arrhenius HPC login account.
2. Accept the NAISS General Terms of Use.
3. Set your Arrhenius password.
4. Register two-factor authentication, usually through a TOTP authenticator app.
5. Confirm that your account status is enabled.

The examples below use `chunqiu` as the Arrhenius username. Replace it with your own username.

## 2. Windows Setup: Use WSL

On Windows, the most stable workflow is usually:

```text
Windows Terminal / PowerShell
        ↓
WSL Ubuntu
        ↓
SSH login to Arrhenius
```

Check WSL in PowerShell:

```powershell
wsl --status
wsl --version
wsl -l -v
```

If you already have a stable Ubuntu installation, you can keep it as a clean base and create a separate daily working copy:

| WSL distribution | Purpose |
| --- | --- |
| `Ubuntu` | Stable base environment, keep mostly unchanged |
| `Ubuntu-work` | Daily SSH, Python, Julia, CUDA-related experiments |

Create the working copy:

```powershell
mkdir C:\WSL_Backup
mkdir D:\WSL
wsl --shutdown
wsl --export Ubuntu C:\WSL_Backup\Ubuntu_base_backup.tar
wsl --import Ubuntu-work D:\WSL\Ubuntu-work C:\WSL_Backup\Ubuntu_base_backup.tar --version 2
wsl -l -v
wsl -d Ubuntu-work
```

If you do not have a `D:` drive, use `C:\WSL\Ubuntu-work` instead.

Imported WSL distributions may start as `root`. If your normal Linux username is `xiach`, edit:

```bash
sudo nano /etc/wsl.conf
```

Add:

```ini
[user]
default=xiach
```

Then restart from PowerShell:

```powershell
wsl --terminate Ubuntu-work
wsl -d Ubuntu-work
```

Check SSH:

```bash
ssh -V
```

If the SSH client is missing:

```bash
sudo apt update
sudo apt install openssh-client -y
```

## 3. macOS Setup

macOS already includes an SSH client. Open Terminal and check:

```bash
ssh -V
```

If you need extra command-line tools, install Homebrew:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install rsync wget git
```

The remaining SSH and file-transfer workflow is essentially the same as Linux/WSL.

## 4. SSH Login to Arrhenius

Basic login:

```bash
ssh chunqiu@login.hpc.arrhenius.naiss.se
```

On the first connection, SSH may ask whether to accept the host fingerprint. If the address is correct, type:

```text
yes
```

Then enter:

```text
password: <your Arrhenius password>
Verification code: <your 2FA code>
```

If the main login host is temporarily unavailable, try:

```bash
ssh chunqiu@arrhenius1.hpc.arrhenius.naiss.se
ssh chunqiu@arrhenius3.hpc.arrhenius.naiss.se
```

### 4.1 Configure SSH Shortcuts

On local WSL or macOS:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/config
```

Add:

```sshconfig
Host arrhenius
    HostName login.hpc.arrhenius.naiss.se
    User chunqiu
    ServerAliveInterval 60
    ServerAliveCountMax 5

Host arrhenius1
    HostName arrhenius1.hpc.arrhenius.naiss.se
    User chunqiu
    ServerAliveInterval 60
    ServerAliveCountMax 5

Host arrhenius3
    HostName arrhenius3.hpc.arrhenius.naiss.se
    User chunqiu
    ServerAliveInterval 60
    ServerAliveCountMax 5
```

Set permissions:

```bash
chmod 600 ~/.ssh/config
```

You can then log in with:

```bash
ssh arrhenius
```

## 5. ThinLinc Remote Desktop

For a graphical desktop, install ThinLinc Client:

```text
https://www.cendio.com/thinlinc/download
```

Open ThinLinc Client and use:

```text
Server: tl.hpc.arrhenius.naiss.se
Username: chunqiu
Password: your Arrhenius password
```

Then enter your 2FA code when requested.

## 6. First Checks After Login

After logging in, run:

```bash
whoami
hostname
pwd
storagequota
projinfo
sinfo
squeue -u $USER
```

These commands confirm your identity, node, working directory, storage quotas, project accounts, Slurm partition state, and your own jobs.

## 7. Storage Layout on Arrhenius

Common locations:

| Location | Example | Recommended use |
| --- | --- | --- |
| Home | `/home/USERNAME` | Small personal config, SSH, lightweight scripts |
| Project storage | `/nobackup/proj/disk/PROJECT_NAME` | Project code, data, environments, results, logs |
| Node-local scratch | `/scratch/local` or `$SNIC_TMP` | Temporary files during a running job |

Important notes:

- Home space is small. Do not use it for large datasets, model caches, or large Python/conda environments.
- Project storage is the main working area, but Arrhenius project directories do not have normal snapshots. Deleted files may be impossible to restore.
- Keep important code on GitHub and regularly download or back up high-value results.
- Use `storagequota` to see the project directories available to you.

Recommended project layout:

```text
/nobackup/proj/disk/<project-name>/users/chunqiu/
├── code/
├── data/
├── envs/
├── jobs/
├── logs/
└── results/
```

## 8. File Transfer

Prefer `rsync` for larger or repeated transfers because it can resume and only transfers changed files.

Upload a local project to home:

```bash
rsync -av ~/Battery_Project/ arrhenius:~/Battery_Project/
```

Upload to project storage:

```bash
ssh arrhenius
storagequota
mkdir -p /nobackup/proj/disk/my_project/users/chunqiu
exit

rsync -av ~/Battery_Project/ arrhenius:/nobackup/proj/disk/my_project/users/chunqiu/Battery_Project/
```

Download results:

```bash
rsync -av arrhenius:/nobackup/proj/disk/my_project/users/chunqiu/Battery_Project/results/ ~/Arrhenius_results/
```

For single files, `scp` is also fine:

```bash
scp local_file.py arrhenius:~/
scp arrhenius:~/remote_file.txt ./
```

## 9. Key Arrhenius Point: ARM-Based GPU Supercomputer Nodes

Arrhenius has `cpu`, `gpu`, and `fat` Slurm partitions. The CPU-only partition uses AMD EPYC x86_64 CPUs. The GPU partition uses NVIDIA GH200 Grace Hopper Superchips, where the CPU side is ARM-based.

This difference matters in practice:

| Where it runs | CPU architecture | Software/container implication |
| --- | --- | --- |
| `cpu` / `fat` partition | x86_64 | Use CPU partition modules and x86_64 binaries |
| `gpu` partition | ARM / `aarch64` + NVIDIA GPU | Use `GPU/` modules; use ARM-compatible wheels, binaries, builds, and containers |

When migrating workflows from Alvis, Tetralith, or Dardel, check:

- whether old Python wheels are x86_64-only
- whether old conda environments contain x86_64 binary packages
- whether old Apptainer/Singularity containers were built for x86_64
- whether your C/C++/Fortran/CUDA extensions must be rebuilt on the GPU partition
- whether GPU jobs load modules with the `GPU/` prefix

Useful checks:

```bash
uname -m
python -c "import platform; print(platform.machine())"
module avail
module avail GPU/
```

If the output is `aarch64`, you are on an ARM environment.

## 10. Lmod Module System

Arrhenius uses Lmod:

```bash
module avail
module avail Python
module avail Miniforge
module avail buildenv
module list
module load <module-name>
module unload <module-name>
module purge
```

GPU-partition software modules usually start with `GPU/`, for example:

```bash
module avail GPU/
module load GPU/buildenv-nvhpc/<version>
```

Important principle: do not mix CPU and GPU partition modules. Modules without the `GPU/` prefix are generally for CPU/x86_64; modules with the `GPU/` prefix are for the GPU/ARM partition.

## 11. Python Environment Notes

Recommended order:

1. First check whether an official Python / Miniforge module is enough.
2. If you need custom dependencies, create the environment in project storage, not in `/home`.
3. Maintain separate CPU and GPU/ARM environments.

CPU partition example:

```bash
cd /nobackup/proj/disk/my_project/users/chunqiu
mkdir -p envs
module purge
module load Python/<available-version>
python -m venv envs/py_cpu
source envs/py_cpu/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install numpy pandas scipy matplotlib
```

For GPU/ARM work, build and test inside a GPU interactive allocation or GPU batch job:

```bash
salloc -A naissXXXX-XX-XXX-gpu -p gpu --gpus=1 -n 1 -c 8 -t 00:30:00

cd /nobackup/proj/disk/my_project/users/chunqiu
module purge
module avail GPU/
module load GPU/<suitable-module>
python -m venv envs/py_gpu_arm
source envs/py_gpu_arm/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -c "import platform; print(platform.machine())"
```

If a package fails to install on the GPU partition, first check whether it supports Linux `aarch64`. For battery work with PyBaMM, NumPy/SciPy, JAX, PyTorch, or similar stacks, pay special attention to Python version, BLAS/LAPACK, CUDA/NVIDIA, and ARM wheel availability.

## 12. Check CPU/GPU Access and Partitions

Check project accounts:

```bash
projinfo
```

Project account names usually indicate the resource type:

| Suffix | Meaning |
| --- | --- |
| `-cpu` | Can submit to CPU resources |
| `-gpu` | Can submit to the `gpu` partition |

Check partitions:

```bash
sinfo
sinfo -o "%P %a %l %D %t %C"
sinfo -p gpu -o "%P %D %t %G %C %m %f"
sinfo -N -p gpu -o "%N %t %G %C %m %f"
```

Request a short interactive GPU session:

```bash
salloc -A naissXXXX-XX-XXX-gpu -p gpu --gpus=1 -n 1 -c 8 -t 00:15:00
nvidia-smi
nvidia-smi -L
uname -m
exit
```

## 13. Slurm Job Submission

Useful commands:

```bash
sinfo
squeue
squeue -u $USER
sbatch job.sh
scancel <job_id>
sacct
```

### 13.1 CPU Job Example

Save as `job_cpu.sh`:

```bash
#!/bin/bash
#SBATCH -A naissXXXX-XX-XXX-cpu
#SBATCH -p cpu
#SBATCH -n 1
#SBATCH -c 4
#SBATCH -t 00:10:00
#SBATCH -J test_cpu
#SBATCH -o logs/%x_%j.out
#SBATCH -e logs/%x_%j.err

set -euo pipefail

cd /nobackup/proj/disk/my_project/users/chunqiu
mkdir -p logs

module purge
module load Python/<available-version>
source envs/py_cpu/bin/activate

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export PYTHONUNBUFFERED=1

hostname
date
python your_script.py
```

Submit:

```bash
sbatch job_cpu.sh
squeue -u $USER
```

### 13.2 GPU/ARM Job Example

Save as `job_gpu.sh`:

```bash
#!/bin/bash
#SBATCH -A naissXXXX-XX-XXX-gpu
#SBATCH -p gpu
#SBATCH -n 1
#SBATCH -c 8
#SBATCH --gpus=1
#SBATCH -t 00:30:00
#SBATCH -J test_gpu_arm
#SBATCH -o logs/%x_%j.out
#SBATCH -e logs/%x_%j.err

set -euo pipefail

cd /nobackup/proj/disk/my_project/users/chunqiu
mkdir -p logs

module purge
module load GPU/<suitable-module>
source envs/py_gpu_arm/bin/activate

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export PYTHONUNBUFFERED=1

hostname
date
uname -m
nvidia-smi
python your_gpu_script.py
```

Submit:

```bash
sbatch job_gpu.sh
squeue -u $USER
```

Remember: the GPU partition CPU architecture is ARM. Do not directly reuse x86_64 virtual environments or containers from Alvis. Recreate environments, rebuild extensions, and rebuild containers on Arrhenius for the target partition.

## 14. Containers and Compiled Software

Arrhenius supports Apptainer/Singularity. Containers are useful for reducing many-small-file pressure on Lustre, but GPU-partition containers must be built for ARM.

When migrating an old container:

```bash
apptainer inspect -d old_container.sif > old_container.def
```

Then rebuild on the target partition:

- Build CPU containers on CPU nodes.
- Build GPU containers on GPU nodes or in GPU jobs.
- GPU partition containers must target `aarch64` / ARM.

A GPU-partition rebuild flow:

```bash
salloc -A naissXXXX-XX-XXX-gpu -p gpu --gpus=1 -n 1 -c 8 -t 01:00:00
uname -m
apptainer build my_gpu_arm_container.sif my_recipe.def
exit
```

The same idea applies to compiled software: compile with the architecture and module stack that match the partition where the program will run.

## 15. VS Code Remote-SSH

If `ssh arrhenius` already works in your terminal, VS Code Remote-SSH can reuse the same local `~/.ssh/config`.

Workflow:

1. Install VS Code locally.
2. Install the `Remote - SSH` extension.
3. Run `Remote-SSH: Connect to Host...`.
4. Select `arrhenius`.
5. Open a project directory such as `/nobackup/proj/disk/my_project/users/chunqiu/code`.

Reminder: VS Code is only an entry point. Large environments, data, and logs should stay in project storage.

## 16. Troubleshooting

### ThinLinc address does not open in a browser

`tl.hpc.arrhenius.naiss.se` is a ThinLinc Client server address. Use ThinLinc Client instead of a browser.

### SSH timeout

Windows PowerShell:

```powershell
Test-NetConnection login.hpc.arrhenius.naiss.se -Port 22
```

Linux/macOS:

```bash
nc -vz login.hpc.arrhenius.naiss.se 22
```

If port 22 is blocked, try a university network, VPN, or mobile hotspot.

### SSH host key warning

Remove the old record carefully:

```bash
ssh-keygen -R login.hpc.arrhenius.naiss.se
ssh arrhenius
```

Only accept the new host key if you are sure the address is correct.

### Project directory is not visible

If you were just added to a project, log out and log back in. Linux group membership is normally initialized at login.

```bash
storagequota
projinfo
```

### GPU Python package fails to install

First check the current architecture:

```bash
uname -m
python -c "import platform; print(platform.machine())"
```

If it is `aarch64`, you need Linux ARM wheels or a local build on the Arrhenius GPU partition. Do not directly copy an x86_64 environment directory and expect it to work.

## 17. Recommended Daily Workflow

Windows:

```powershell
wsl -d Ubuntu-work
```

Inside WSL:

```bash
ssh arrhenius
```

On Arrhenius:

```bash
storagequota
projinfo
sinfo
squeue -u $USER
```

For first-time GPU work, start with a short interactive allocation and inspect the architecture and GPU:

```bash
salloc -A naissXXXX-XX-XXX-gpu -p gpu --gpus=1 -n 1 -c 8 -t 00:15:00
uname -m
nvidia-smi
module avail GPU/
exit
```

## 18. Official References

- NAISS Arrhenius overview: <https://www.naiss.se/resource/arrhenius/>
- NAISS Arrhenius technical description: <https://www.naiss.se/resources/arrhenius-technical-description/>
- Arrhenius HPC Quick Start: <https://hpc.pages.naiss.se/user-documentation/support-docs/arrhenius_hpc/basics/quickstart/>
