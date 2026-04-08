# Alvis Environment Setup and Usage Guide

This document is intended for group members working on C3SE Alvis. After reading it, you should be able to:

- understand the difference between your personal home area and your project storage
- use Alvis public environments first instead of installing everything yourself
- load Python, CUDA, and other software through `module load`
- run CPU or GPU jobs through `sbatch`
- connect your existing SSH access to VS Code Remote-SSH
- expose your own environment in the web Jupyter entry
- understand why PyBaMM work in our battery projects needs a separate environment

## 1. Two Different Storage Areas on Alvis

On Alvis, your personal home directory and your project storage are not the same thing.

- Personal area: usually your home directory after login, for example `~/`, mapped locally to something like `/cephyr/users/<username>/Alvis`
- Project area: the larger storage allocated to the project, for example `/mimer/NOBACKUP/groups/naiss2025-22-468`

Recommended split:

- Keep these in your personal area:
  - `~/.ssh/config`
  - `~/.bashrc`
  - `~/portal/jupyter/*.sh`
  - small scripts and lightweight configuration
- Keep these in the project area:
  - virtual environments
  - self-installed software
  - datasets
  - training outputs and logs
  - `pip` caches and model caches

The reason is straightforward: personal home space is usually limited, while project storage is much better suited for large environments, caches, and results.

Example:

```bash
cd /mimer/NOBACKUP/groups/naiss2025-22-468
mkdir -p envs logs tmp
```

You may also want to redirect caches into the project area:

```bash
export PIP_CACHE_DIR=/mimer/NOBACKUP/groups/naiss2025-22-468/.cache/pip
export HF_HOME=/mimer/NOBACKUP/groups/naiss2025-22-468/.cache/huggingface
export MPLCONFIGDIR=/mimer/NOBACKUP/groups/naiss2025-22-468/.cache/matplotlib
```

## 2. Use Alvis Public Environments First

If your work does not depend on special legacy package versions, the best starting point is the public software stack already provided by Alvis.

Check available modules:

```bash
module avail Python
module avail CUDA
module spider PyTorch
```

Typical usage:

```bash
module purge
module load Python/3.12.3-GCCcore-13.3.0
python --version
which python
```

If you also need GPU support:

```bash
module purge
module load Python/3.12.3-GCCcore-13.3.0
module load CUDA/12.6.0
python --version
nvcc --version
```

Advantages of public environments:

- no local installation is needed
- they are usually well integrated with the cluster
- they are sufficient for many standard Python workflows

Recommended order:

1. Check whether the public environment is enough
2. If yes, use `module load`
3. Only build your own environment if version compatibility requires it

## 3. Why PyBaMM Needs a Separate Environment for Battery Work

For our battery projects, especially when PyBaMM is required, relying on the latest system Python is often not enough.

Why:

- PyBaMM can be sensitive to the versions of `numpy`, `scipy`, and related libraries
- our `requirements.txt` often expects a more specific dependency combination
- newer system Python versions are not always compatible with that stack

Because of this, the safer workflow is:

- load a lower and more compatible Python module first as the base interpreter
- then create and install our own Python environment in the project area, for example `bamm_venv`
- install PyBaMM and all required libraries from `requirements.txt`

This is important: install the environment in the project storage, not in your small personal area.

One point is worth making very clearly: `module load Python/...` does not mean the battery environment is already installed. It only gives us a compatible lower-version Python from Alvis so that we can build our own environment on top of it. The actual working environment is still the one we install ourselves in the project area, such as `bamm_venv` or `bamm_venv_gpu`.

Example:

```bash
cd /mimer/NOBACKUP/groups/naiss2025-22-468
module purge
module load Python/3.11.3-GCCcore-12.3.0
python -m venv bamm_venv
source bamm_venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install -r /path/to/requirements.txt
```

What these steps mean:

- `module load Python/3.11.3-GCCcore-12.3.0`: use a compatible lower-version Python provided by Alvis
- `python -m venv bamm_venv`: create our own environment in the project storage
- `pip install -r requirements.txt`: actually install PyBaMM and the required versions of `numpy`, `scipy`, and other dependencies into that environment

If you prefer a cleaner layout, you can also do this:

```bash
cd /mimer/NOBACKUP/groups/naiss2025-22-468
mkdir -p envs

module purge
module load Python/3.11.3-GCCcore-12.3.0
python -m venv envs/bamm_venv
source envs/bamm_venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install -r /path/to/requirements.txt
```

To add extra packages later:

```bash
module purge
module load Python/3.11.3-GCCcore-12.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/bamm_venv/bin/activate
python -m pip install seaborn
python -m pip install jupyterlab
python -m pip install <your-package>
```

If you also need GPU support later, you can build a separate `bamm_venv_gpu`. In practice this usually starts from a public GPU-enabled module rather than a plain `Python/...` module, and then layers our own packages on top. For example:

```bash
cd /mimer/NOBACKUP/groups/naiss2025-22-468
module purge
module load PyTorch/2.1.2-foss-2023a-CUDA-12.1.1
python -m venv --system-site-packages bamm_venv_gpu
source bamm_venv_gpu/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install -r /path/to/requirements.txt
python -m pip install <your-extra-packages>
```

The idea behind this pattern is:

- the public module provides a compatible GPU / CUDA / Python runtime stack
- `bamm_venv_gpu` then adds our own PyBaMM and project-specific dependencies
- this lets us reuse the Alvis GPU software stack while still controlling our own package versions

If your workflow does not actually need GPU acceleration, maintaining only `bamm_venv` is enough.

## 4. Very Important Detail: `module load` First, Then Activate the venv

On Alvis, a `venv` is usually tied to the system Python module that was used when it was created.

So this is often not sufficient:

```bash
source /mimer/NOBACKUP/groups/naiss2025-22-468/bamm_venv/bin/activate
python script.py
```

The safer pattern is:

```bash
module purge
module load Python/3.11.3-GCCcore-12.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/bamm_venv/bin/activate
python script.py
```

In other words:

- the `venv` was built on top of a specific module version
- load the same Python module before using it
- then activate the environment

Otherwise you may hit errors such as missing `libpython*.so`.

## 5. Recommended Environment Layout

A clean structure is usually easier to maintain:

```text
/mimer/NOBACKUP/groups/naiss2025-22-468/
├── envs/
│   ├── py311_pybamm/
│   └── py311_pybamm_gpu/
├── datasets/
├── logs/
├── software/
└── tmp/
```

Example:

```bash
cd /mimer/NOBACKUP/groups/naiss2025-22-468
mkdir -p envs datasets logs software tmp

module purge
module load Python/3.11.3-GCCcore-12.3.0
python -m venv envs/py311_pybamm
source envs/py311_pybamm/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install -r /path/to/requirements.txt
```

## 6. Run Commands with `sbatch`

On Alvis, production jobs should normally be submitted through `sbatch` instead of running for a long time on the login node.

### 6.1 CPU job example

If the job is CPU-only, it is better to request `NOGPU` explicitly instead of leaving the GPU type unspecified. This matches the intended usage of the CPU-only nodes on Alvis and avoids making the script look like a generic job on GPU-capable nodes.

Save this as `run_cpu.sbatch`:

```bash
#!/bin/bash
#SBATCH -A naiss2025-22-468
#SBATCH -p alvis
#SBATCH -C NOGPU
#SBATCH -J pybamm_cpu
#SBATCH -N 1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH -t 04:00:00
#SBATCH -o logs/%x_%j.out
#SBATCH -e logs/%x_%j.err

set -euo pipefail

cd /mimer/NOBACKUP/groups/naiss2025-22-468

module purge
module load Python/3.11.3-GCCcore-12.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/envs/py311_pybamm/bin/activate

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export NUMEXPR_MAX_THREADS=${SLURM_CPUS_PER_TASK}

srun --cpu-bind=cores python your_script.py
```

Submit:

```bash
sbatch run_cpu.sbatch
```

Check queue:

```bash
squeue -u $USER
```

### 6.2 GPU job example

If your workflow needs GPU resources, the following pattern is closer to what we actually use in practice. It is based on scripts such as:

`/mimer/NOBACKUP/groups/naiss2026-4-521/PulseBat/gpu_combo_all/run_gpu_combo_all_lfp3_fai_irrev_bad_crate_rerun.sbatch`

Save this as `run_gpu.sbatch`:

```bash
#!/bin/bash
#SBATCH -A naiss2025-22-468
#SBATCH -p alvis
#SBATCH -J pybamm_gpu
#SBATCH -N 1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --gpus-per-node=T4:1
#SBATCH -t 08:00:00
#SBATCH -o logs/%x_%j.out
#SBATCH -e logs/%x_%j.err

set -euo pipefail

cd /mimer/NOBACKUP/groups/naiss2025-22-468

module purge
module load PyTorch/2.1.2-foss-2023a-CUDA-12.1.1
source /mimer/NOBACKUP/groups/naiss2025-22-468/bamm_venv_gpu/bin/activate

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK}
export NUMEXPR_MAX_THREADS=${SLURM_CPUS_PER_TASK}
export PYTHONUNBUFFERED=1
export MPLBACKEND=Agg

nvidia-smi
srun --cpu-bind=cores python your_gpu_script.py
```

Notes:

- for CPU-only jobs, explicitly add `#SBATCH -C NOGPU`; this is the recommended way to target the no-GPU nodes in the C3SE Alvis documentation
- `module load PyTorch/...` here does not mean your code must be PyTorch-based; it is often used because that public module already provides a compatible GPU/CUDA runtime stack, and we then layer our own `bamm_venv_gpu` on top of it
- if your GPU environment was built against a different public module stack, replace this module line accordingly
- common GPU types on Alvis include `A100`, `A100fat`, `A40`, `V100`, and `T4`
- if you need a specific GPU type, use something like `#SBATCH --gpus-per-node=A100:1` or `#SBATCH --gpus-per-node=T4:1`
- to check available GPU types, hardware details, and current queue/resource status, use the official C3SE Alvis page: `https://www.c3se.chalmers.se/about/Alvis/`
- according to the C3SE page, the live `Queue` view is typically only accessible from within SUNET networks, so off campus you usually need VPN
- from the command line on Alvis, `jobinfo`, `squeue`, and `sinfo` are also useful for checking queue and resource availability

## 7. Connect Alvis to VS Code Through SSH

If you can already log in from a local terminal with SSH, connecting through VS Code Remote-SSH is usually straightforward.

### 7.1 Local SSH config

Add something like this to `~/.ssh/config` on your own computer:

```sshconfig
Host alvis
    HostName <your-alvis-login-host>
    User <your-cid>
    IdentityFile ~/.ssh/alvis1_local
```

Explanation:

- `Host alvis` is your own alias
- replace `HostName` with the actual Alvis login host provided by C3SE
- replace `User` with your own username
- replace `IdentityFile` with the correct path to your private key

If this already works in a terminal:

```bash
ssh alvis
```

then VS Code can reuse the same SSH alias directly.

### 7.2 VS Code steps

1. Install VS Code locally
2. Install the `Remote - SSH` extension
3. Open the command palette and run `Remote-SSH: Connect to Host...`
4. Select the `alvis` host entry
5. Wait for VS Code to install its remote server on Alvis
6. Open the working directory on Alvis

Recommended directories to open:

- your code directory
- your project work directory
- for example `/mimer/NOBACKUP/groups/naiss2025-22-468`

Reminder:

- SSH configuration such as `~/.ssh/config` belongs in your own home area or local machine
- it is not something to place inside the shared project storage

## 8. Jupyter Notebook / JupyterLab in the Web Portal

In the Alvis web portal, you can make your own environment appear in the Jupyter choices by adding shell scripts under `~/portal/jupyter/*.sh`.

For example, you already have:

```bash
/cephyr/users/chunqiu/Alvis/portal/jupyter/py312_bamm_seaborn.sh
```

Each user should replace the username in that path with their own username.

A typical script looks like this:

```bash
ml purge
ml Python/3.12.3-GCCcore-13.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/bamm_venv/bin/activate
jupyter lab --config="${CONFIG_FILE}" --FileCheckpoints.checkpoint_dir='/mimer/NOBACKUP/groups/naiss2025-22-468/.ipynb_uni'
```

What it does:

- clears the current module environment
- loads the Python module that matches the virtual environment
- activates the `bamm_venv` stored in the project area
- launches JupyterLab in the web portal
- stores notebook checkpoints in the project area

### 8.1 Add your own Jupyter entry

Example:

```bash
mkdir -p ~/portal/jupyter
cp ~/portal/jupyter/py312_bamm_seaborn.sh ~/portal/jupyter/py311_pybamm.sh
chmod +x ~/portal/jupyter/py311_pybamm.sh
```

Then edit it to match your environment, for example:

```bash
ml purge
ml Python/3.11.3-GCCcore-12.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/envs/py311_pybamm/bin/activate
jupyter lab --config="${CONFIG_FILE}" --FileCheckpoints.checkpoint_dir='/mimer/NOBACKUP/groups/naiss2025-22-468/.ipynb_uni'
```

This makes it easier to find and launch your own environment from the portal Jupyter menu.

## 9. Recommended Onboarding Flow for New Group Members

If this is your first time on Alvis, the following order usually works well:

1. learn SSH login and VS Code Remote-SSH first
2. learn `module purge` and `module load`
3. check whether the public environment is enough
4. if the project really needs PyBaMM and pinned versions, build a `venv` in the project area
5. submit real jobs through `sbatch`
6. only then add a custom launcher under `~/portal/jupyter/` if you want notebook access in the web portal

## 10. Quick Command Reference

```bash
# Check modules
module avail Python
module avail CUDA
module spider PyTorch

# Load public Python
module purge
module load Python/3.12.3-GCCcore-13.3.0

# Create a PyBaMM environment
cd /mimer/NOBACKUP/groups/naiss2025-22-468
module purge
module load Python/3.11.3-GCCcore-12.3.0
python -m venv envs/py311_pybamm
source envs/py311_pybamm/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install -r /path/to/requirements.txt

# Activate an existing environment
module purge
module load Python/3.11.3-GCCcore-12.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/envs/py311_pybamm/bin/activate

# Submit jobs
sbatch run_cpu.sbatch
sbatch run_gpu.sbatch

# Check jobs
squeue -u $USER
```

## 11. Final Advice

- when unsure, try the public environment first
- if you need your own environment, put it in the project storage
- for PyBaMM and other version-sensitive stacks, pin Python and `requirements.txt`
- before using a `venv`, load the same Python module it was built with
- for real compute jobs, prefer `sbatch`
- VS Code and Jupyter are only entry points; the large environments, caches, data, and logs should live in the project area
