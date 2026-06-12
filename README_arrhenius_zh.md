# Arrhenius 环境配置与使用说明

本文面向准备从 Windows 或 macOS 访问 NAISS Arrhenius 的组内同学，目标是让大家能快速完成登录、文件传输、环境检查、软件模块使用和 Slurm 作业提交。

Arrhenius 的重点变化是：GPU 分区不是传统 x86 GPU 节点，而是 NVIDIA GH200 Grace Hopper Superchip 节点。每个 GPU 节点包含 ARM 架构的 Grace CPU 和 Hopper GPU。因此，凡是要在 `gpu` 分区运行的程序、Python 包、编译产物和容器，都要特别注意 `aarch64` / ARM 架构兼容性。

## 0. 快速概览

| 事项 | 推荐方式 |
| --- | --- |
| 命令行登录 | SSH |
| Windows 本地环境 | Windows Terminal / PowerShell + WSL Ubuntu |
| macOS 本地环境 | Terminal |
| 文件传输 | `rsync`、`scp`、`sftp`、FileZilla |
| 远程桌面 | ThinLinc Client |
| 账户、项目和资源 | SUPR |
| 计算作业 | Slurm：`sbatch`、`salloc`、`squeue`、`sinfo` |

常用地址：

```text
SSH:       login.hpc.arrhenius.naiss.se
Fallback:  arrhenius1.hpc.arrhenius.naiss.se
Fallback:  arrhenius3.hpc.arrhenius.naiss.se
ThinLinc:  tl.hpc.arrhenius.naiss.se
```

> `tl.hpc.arrhenius.naiss.se` 是 ThinLinc Client 里填写的服务器地址，不一定能作为普通网页在浏览器中打开。

## 1. 使用前准备

在配置本地电脑之前，请先确认你已经在 SUPR 中完成：

1. 申请 Arrhenius HPC 登录账户。
2. 接受 NAISS General Terms of Use。
3. 设置 Arrhenius 密码。
4. 注册两步验证，通常是 TOTP 验证器应用。
5. 确认账户状态已经启用。

下面命令里的用户名以 `chunqiu` 为例。实际使用时请替换成自己的 Arrhenius 用户名。

## 2. Windows 推荐配置：通过 WSL 访问

Windows 上最稳的工作流通常是：

```text
Windows Terminal / PowerShell
        ↓
WSL Ubuntu
        ↓
SSH 登录 Arrhenius
```

先在 PowerShell 检查 WSL：

```powershell
wsl --status
wsl --version
wsl -l -v
```

如果你已经有一个稳定的 Ubuntu，可以保留它作为基础环境，再复制一份日常工作环境，例如：

| WSL 发行版 | 用途 |
| --- | --- |
| `Ubuntu` | 稳定基础环境，尽量少动 |
| `Ubuntu-work` | 日常 SSH、Python、Julia、CUDA 相关尝试 |

创建工作副本：

```powershell
mkdir C:\WSL_Backup
mkdir D:\WSL
wsl --shutdown
wsl --export Ubuntu C:\WSL_Backup\Ubuntu_base_backup.tar
wsl --import Ubuntu-work D:\WSL\Ubuntu-work C:\WSL_Backup\Ubuntu_base_backup.tar --version 2
wsl -l -v
wsl -d Ubuntu-work
```

如果没有 `D:` 盘，把路径换成 `C:\WSL\Ubuntu-work`。

导入后的 WSL 可能会以 `root` 启动。假设你的普通 Linux 用户是 `xiach`，可以在 WSL 中写入：

```bash
sudo nano /etc/wsl.conf
```

内容：

```ini
[user]
default=xiach
```

然后回到 PowerShell：

```powershell
wsl --terminate Ubuntu-work
wsl -d Ubuntu-work
```

检查 SSH：

```bash
ssh -V
```

如果缺少 SSH 客户端：

```bash
sudo apt update
sudo apt install openssh-client -y
```

## 3. macOS 配置

macOS 自带 SSH 客户端，打开 Terminal 后检查：

```bash
ssh -V
```

如果需要额外工具，可以安装 Homebrew：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install rsync wget git
```

后续 SSH 和文件传输流程与 Linux/WSL 基本一致。

## 4. SSH 登录 Arrhenius

基础登录：

```bash
ssh chunqiu@login.hpc.arrhenius.naiss.se
```

首次连接时，SSH 可能会询问是否接受主机指纹。确认地址正确后输入：

```text
yes
```

然后依次输入：

```text
password: <your Arrhenius password>
Verification code: <your 2FA code>
```

如果主登录地址暂时不可用，可以试：

```bash
ssh chunqiu@arrhenius1.hpc.arrhenius.naiss.se
ssh chunqiu@arrhenius3.hpc.arrhenius.naiss.se
```

### 4.1 配置 SSH 快捷方式

在本地 WSL 或 macOS 中：

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/config
```

加入：

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

设置权限：

```bash
chmod 600 ~/.ssh/config
```

之后可以直接：

```bash
ssh arrhenius
```

## 5. ThinLinc 远程桌面

如果需要图形界面，安装 ThinLinc Client：

```text
https://www.cendio.com/thinlinc/download
```

打开 ThinLinc Client 后填写：

```text
Server: tl.hpc.arrhenius.naiss.se
Username: chunqiu
Password: your Arrhenius password
```

随后按提示输入 2FA 验证码。

## 6. 登录后的第一组检查

登录后先运行：

```bash
whoami
hostname
pwd
storagequota
projinfo
sinfo
squeue -u $USER
```

这些命令分别用于确认身份、当前节点、当前位置、存储配额、可用项目账户、Slurm 分区状态和自己的队列。

## 7. Arrhenius 存储布局

常见位置：

| 位置 | 示例 | 推荐用途 |
| --- | --- | --- |
| Home | `/home/USERNAME` | 小型个人配置、SSH、少量脚本 |
| Project storage | `/nobackup/proj/disk/PROJECT_NAME` | 项目代码、数据、环境、结果、日志 |
| Node-local scratch | `/scratch/local` 或 `$SNIC_TMP` | 作业运行期间的临时文件 |

重要提醒：

- Home 空间较小，不适合放大数据集、模型缓存或大型 Python/conda 环境。
- 项目目录是主要工作位置，但 Arrhenius 项目目录没有普通快照；删除的文件可能无法恢复。
- 重要代码建议放 GitHub，关键结果也应定期下载或备份。
- 用 `storagequota` 查看自己能访问的项目目录。

推荐项目目录结构：

```text
/nobackup/proj/disk/<project-name>/users/chunqiu/
├── code/
├── data/
├── envs/
├── jobs/
├── logs/
└── results/
```

## 8. 文件传输

推荐优先使用 `rsync`，因为它可以断点续传，并且只同步变化的文件。

上传本地项目到 home：

```bash
rsync -av ~/Battery_Project/ arrhenius:~/Battery_Project/
```

上传到项目目录：

```bash
ssh arrhenius
storagequota
mkdir -p /nobackup/proj/disk/my_project/users/chunqiu
exit

rsync -av ~/Battery_Project/ arrhenius:/nobackup/proj/disk/my_project/users/chunqiu/Battery_Project/
```

下载结果：

```bash
rsync -av arrhenius:/nobackup/proj/disk/my_project/users/chunqiu/Battery_Project/results/ ~/Arrhenius_results/
```

单文件也可以用 `scp`：

```bash
scp local_file.py arrhenius:~/
scp arrhenius:~/remote_file.txt ./
```

## 9. Arrhenius 的核心重点：ARM 架构 GPU 超算节点

Arrhenius 有 CPU、GPU、fat 等 Slurm 分区。CPU-only 分区是 AMD EPYC x86_64；GPU 分区是 NVIDIA GH200 Grace Hopper Superchip，CPU 侧是 ARM 架构。

这会带来一个很实际的区别：

| 运行位置 | CPU 架构 | 软件/容器注意事项 |
| --- | --- | --- |
| `cpu` / `fat` 分区 | x86_64 | 使用普通 CPU 分区模块和 x86_64 二进制 |
| `gpu` 分区 | ARM / `aarch64` + NVIDIA GPU | 使用 `GPU/` 模块；需要 ARM 兼容 wheel、二进制、编译产物和容器 |

因此迁移 Alvis/Tetralith/Dardel 上的旧工作流时，请重点检查：

- 旧的 Python wheel 是否只支持 x86_64。
- 旧 conda 环境是否包含 x86_64 二进制包。
- 旧 Apptainer/Singularity 容器是否为 x86_64 构建。
- 自己编译的 C/C++/Fortran/CUDA 扩展是否需要在 GPU 分区重新编译。
- 运行 GPU 作业时是否加载了 `GPU/` 前缀模块。

常用检查命令：

```bash
uname -m
python -c "import platform; print(platform.machine())"
module avail
module avail GPU/
```

如果输出是 `aarch64`，说明当前环境是 ARM 架构。

## 10. Lmod 模块系统

Arrhenius 使用 Lmod：

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

GPU 分区的软件模块通常以 `GPU/` 开头，例如：

```bash
module avail GPU/
module load GPU/buildenv-nvhpc/<version>
```

一个关键原则：CPU 分区模块和 GPU 分区模块不能混用。没有 `GPU/` 前缀的模块通常用于 CPU/x86_64；带 `GPU/` 前缀的模块用于 GPU/ARM 分区。

## 11. Python 环境建议

优先顺序：

1. 先看官方模块里是否有合适的 Python / Miniforge。
2. 如果需要自定义依赖，把环境建到项目目录，不要塞进 `/home`。
3. CPU 和 GPU/ARM 环境分开维护，不要混用。

CPU 分区示例：

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

GPU/ARM 分区建议在 GPU 交互作业或 GPU batch job 中构建和测试：

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

如果某个包在 GPU/ARM 上安装失败，优先确认它是否支持 Linux `aarch64`。对电池方向的 PyBaMM、NumPy/SciPy、JAX/PyTorch 等栈，要特别注意 Python 版本、底层 BLAS/LAPACK、CUDA/NVIDIA 栈是否匹配。

## 12. CPU/GPU 权限和分区检查

查看项目账户：

```bash
projinfo
```

项目账户通常会体现资源类型：

| 后缀 | 含义 |
| --- | --- |
| `-cpu` | 可提交到 `cpu` 和通常的 CPU 资源 |
| `-gpu` | 可提交到 `gpu` 分区 |

查看分区：

```bash
sinfo
sinfo -o "%P %a %l %D %t %C"
sinfo -p gpu -o "%P %D %t %G %C %m %f"
sinfo -N -p gpu -o "%N %t %G %C %m %f"
```

申请一个短的 GPU 交互会话：

```bash
salloc -A naissXXXX-XX-XXX-gpu -p gpu --gpus=1 -n 1 -c 8 -t 00:15:00
nvidia-smi
nvidia-smi -L
uname -m
exit
```

## 13. Slurm 作业提交

常用命令：

```bash
sinfo
squeue
squeue -u $USER
sbatch job.sh
scancel <job_id>
sacct
```

### 13.1 CPU 作业示例

保存为 `job_cpu.sh`：

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

提交：

```bash
sbatch job_cpu.sh
squeue -u $USER
```

### 13.2 GPU/ARM 作业示例

保存为 `job_gpu.sh`：

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

提交：

```bash
sbatch job_gpu.sh
squeue -u $USER
```

注意：GPU 分区的 CPU 是 ARM 架构，所以不要直接复用 Alvis 上 x86_64 的虚拟环境或容器。最稳妥的做法是在 Arrhenius 的 GPU 分区重新建环境、重新编译扩展、重新构建容器。

## 14. 容器与编译

Arrhenius 支持 Apptainer/Singularity。容器能减少 Lustre 上大量小文件带来的压力，但 GPU 分区必须考虑 ARM 架构。

迁移旧容器时：

```bash
apptainer inspect -d old_container.sif > old_container.def
```

然后在目标分区重新构建：

- CPU 分区要在 CPU 节点构建。
- GPU 分区要在 GPU 节点或 GPU 作业中构建。
- GPU 分区容器需要面向 `aarch64` / ARM。

一个 GPU 分区构建流程可以是：

```bash
salloc -A naissXXXX-XX-XXX-gpu -p gpu --gpus=1 -n 1 -c 8 -t 01:00:00
uname -m
apptainer build my_gpu_arm_container.sif my_recipe.def
exit
```

编译型软件也遵循同样原则：在哪个分区运行，就尽量在哪个分区对应的架构和模块栈中编译。

## 15. VS Code Remote-SSH

如果终端中已经可以 `ssh arrhenius`，VS Code Remote-SSH 可以直接复用本地 `~/.ssh/config`。

流程：

1. 本地安装 VS Code。
2. 安装 `Remote - SSH` 扩展。
3. 执行 `Remote-SSH: Connect to Host...`。
4. 选择 `arrhenius`。
5. 打开项目目录，例如 `/nobackup/proj/disk/my_project/users/chunqiu/code`。

提醒：VS Code 是入口，真正的大环境、数据和日志仍然应放在项目目录。

## 16. 常见问题

### ThinLinc 地址在浏览器打不开

`tl.hpc.arrhenius.naiss.se` 是 ThinLinc Client 的服务器地址。请使用 ThinLinc Client，而不是浏览器。

### SSH 超时

Windows PowerShell：

```powershell
Test-NetConnection login.hpc.arrhenius.naiss.se -Port 22
```

Linux/macOS：

```bash
nc -vz login.hpc.arrhenius.naiss.se 22
```

如果 22 端口被网络阻断，尝试学校网络、VPN 或手机热点。

### SSH host key 警告

谨慎删除旧记录：

```bash
ssh-keygen -R login.hpc.arrhenius.naiss.se
ssh arrhenius
```

只有确认连接地址正确时才接受新的 host key。

### 项目目录看不到

如果刚加入项目，退出重新登录。Linux 组权限通常在登录时初始化。

```bash
storagequota
projinfo
```

### GPU Python 包装不上

先确认当前架构：

```bash
uname -m
python -c "import platform; print(platform.machine())"
```

如果是 `aarch64`，需要找支持 Linux ARM 的 wheel，或者在 Arrhenius GPU 分区本地编译。不要把 x86_64 的环境目录直接搬过来用。

## 17. 推荐日常流程

Windows：

```powershell
wsl -d Ubuntu-work
```

WSL 内：

```bash
ssh arrhenius
```

Arrhenius 上：

```bash
storagequota
projinfo
sinfo
squeue -u $USER
```

第一次做 GPU 工作时，建议先申请短交互会话，检查架构和 GPU：

```bash
salloc -A naissXXXX-XX-XXX-gpu -p gpu --gpus=1 -n 1 -c 8 -t 00:15:00
uname -m
nvidia-smi
module avail GPU/
exit
```

## 18. 官方参考

- NAISS Arrhenius 概览：<https://www.naiss.se/resource/arrhenius/>
- NAISS Arrhenius 技术说明：<https://www.naiss.se/resources/arrhenius-technical-description/>
- Arrhenius HPC Quick Start：<https://hpc.pages.naiss.se/user-documentation/support-docs/arrhenius_hpc/basics/quickstart/>
