# Alvis 环境配置与使用说明

本文面向在 C3SE 的 Alvis 平台上开展计算工作的组内同学，目标是让大家看完后能快速完成以下几件事：

- 理解 Alvis 上个人空间和项目组空间的区别
- 优先使用平台提供的公共环境，而不是一上来就自己装 Python
- 知道如何用 `module load` 调用 Alvis 上已有的 Python / CUDA / 软件模块
- 知道如何通过 `sbatch` 提交 CPU 或 GPU 作业
- 知道如何把自己已有的 SSH 登录方式接到 VS Code Remote-SSH
- 知道如何在网页端 Jupyter Notebook / JupyterLab 中挂载自己的环境
- 知道为什么电池方向的 PyBaMM 环境需要单独处理

## 1. Alvis 上的两类空间

在 Alvis 上，个人家目录和项目申请得到的组空间不是一回事。

- 个人空间：通常是登录后的家目录，例如 `~/`，在本机上对应类似 `/cephyr/users/<username>/Alvis`
- 项目组空间：通常是项目申请得到的大空间，例如 `/mimer/NOBACKUP/groups/naiss2025-22-468`

建议把这两类空间分工使用：

- 放在个人空间里的内容：
  - `~/.ssh/config`
  - `~/.bashrc`
  - `~/portal/jupyter/*.sh`
  - 少量脚本、轻量配置
- 放在项目组空间里的内容：
  - 虚拟环境 `venv`
  - 自己安装的软件
  - 数据集
  - 训练输出、日志
  - `pip` / 模型 / 缓存目录

原因很简单：个人空间通常比较小，不适合长期堆环境和缓存。大一些的环境、数据和结果，建议直接放到项目组空间里。

例如：

```bash
cd /mimer/NOBACKUP/groups/naiss2025-22-468
mkdir -p envs logs tmp
```

如果你有自己的缓存类目录，也建议定向到组空间，例如：

```bash
export PIP_CACHE_DIR=/mimer/NOBACKUP/groups/naiss2025-22-468/.cache/pip
export HF_HOME=/mimer/NOBACKUP/groups/naiss2025-22-468/.cache/huggingface
export MPLCONFIGDIR=/mimer/NOBACKUP/groups/naiss2025-22-468/.cache/matplotlib
```

## 2. 优先使用 Alvis 提供的公共环境

如果你的工作不依赖特殊的旧版本包，最推荐的做法是先用 Alvis 已经装好的公共环境。

先查看可用模块：

```bash
module avail Python
module avail CUDA
module spider PyTorch
```

常见用法是先清理环境，再加载需要的模块：

```bash
module purge
module load Python/3.12.3-GCCcore-13.3.0
python --version
which python
```

如果还需要 GPU 相关环境，可以继续加载 CUDA：

```bash
module purge
module load Python/3.12.3-GCCcore-13.3.0
module load CUDA/12.6.0
python --version
nvcc --version
```

公共环境的优点：

- 不需要自己编译或安装
- 更容易和集群系统兼容
- 适合一般 Python 脚本、数据处理、常规计算任务

建议的工作顺序是：

1. 先看公共环境够不够用
2. 如果够用，直接 `module load`
3. 只有当项目依赖版本确实不兼容时，再自己建环境

## 3. 为什么电池方向的 PyBaMM 要单独建环境

对我们电池方向的工作，尤其是需要使用 PyBaMM 的情况，通常不能只依赖系统里当前默认的高版本 Python。

原因是：

- PyBaMM 对 `numpy`、`scipy` 以及一部分依赖版本比较敏感
- 我们实际使用的 `requirements.txt` 往往需要更特定的版本组合
- 系统里较新的 Python 版本不一定和这套依赖兼容

因此，电池方向更稳妥的做法是：

- 先加载一个较低版本、兼容性更好的 Python 模块作为“底座”
- 再在项目组空间里创建并安装我们自己的 Python 环境，例如 `bamm_venv`
- 再按 `requirements.txt` 安装 PyBaMM 及相关依赖

这里特别强调：环境请尽量装在项目组空间，不要装在个人小空间里。

这里要特别说明一下：`module load Python/...` 并不等于已经有了我们电池方向真正需要的环境。它只是先把 Alvis 上一个兼容的低版本 Python 调出来，供我们后续创建自己的 `venv` 并安装包。真正工作的环境，仍然是我们自己在项目组空间里装出来的，例如 `bamm_venv` 或 `bamm_venv_gpu`。

例如：

```bash
cd /mimer/NOBACKUP/groups/naiss2025-22-468
module purge
module load Python/3.11.3-GCCcore-12.3.0
python -m venv bamm_venv
source bamm_venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install -r /path/to/requirements.txt
```

上面这几步的含义是：

- `module load Python/3.11.3-GCCcore-12.3.0`：先借用 Alvis 上兼容的低版本 Python
- `python -m venv bamm_venv`：在项目组空间里创建我们自己的环境
- `pip install -r requirements.txt`：把 PyBaMM 以及对应版本的 `numpy`、`scipy` 等依赖真正安装进去

如果你想把环境放得更规整，也可以这样：

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

如果之后还要补装自己的包，可以直接：

```bash
module purge
module load Python/3.11.3-GCCcore-12.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/bamm_venv/bin/activate
python -m pip install seaborn
python -m pip install jupyterlab
python -m pip install <your-package>
```

如果你后续还需要 GPU 相关支持，也可以单独再建一个 `bamm_venv_gpu`。这种环境通常不是从纯 `Python/...` 模块起步，而是先加载一个已经带好 CUDA 运行时的公共 GPU 模块，再在其基础上叠加我们自己的包。例如：

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

这套写法的思路是：

- 公共模块提供兼容的 GPU / CUDA / Python 运行时
- `bamm_venv_gpu` 里再补我们自己的 PyBaMM 和项目依赖
- 这样既能复用 Alvis 上现成的 GPU 栈，也能保留我们自己需要的包版本

如果你的任务本身不依赖 GPU，就只维护 `bamm_venv` 即可，不必额外建 `bamm_venv_gpu`。

## 4. 一个非常重要的细节：先 `module load`，再激活 venv

在 Alvis 上，用 `venv` 建出来的环境通常依赖你创建它时对应的系统 Python 模块。

这意味着下面这种写法往往不够：

```bash
source /mimer/NOBACKUP/groups/naiss2025-22-468/bamm_venv/bin/activate
python script.py
```

更稳妥的写法应该是：

```bash
module purge
module load Python/3.11.3-GCCcore-12.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/bamm_venv/bin/activate
python script.py
```

也就是说：

- `venv` 是建立在某个模块版本之上的
- 运行时应先加载同一个 Python 模块
- 然后再 `source .../bin/activate`

否则可能会遇到类似 `libpython*.so` 找不到的错误。

## 5. 推荐的环境组织方式

比较推荐的目录结构如下：

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

例如你可以这样创建：

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

## 6. 用 `sbatch` 运行命令

在 Alvis 上，正式计算任务通常建议通过 `sbatch` 提交，而不是长时间直接在登录节点运行。

### 6.1 CPU 作业示例

如果任务是纯 CPU 计算，建议显式申请 `NOGPU`，不要什么卡型都不写。这样更符合 Alvis 上 CPU-only 节点的使用方式，也能避免让调度器把任务当成普通 GPU 节点上的作业去处理。

保存为 `run_cpu.sbatch`：

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

提交：

```bash
sbatch run_cpu.sbatch
```

查看队列：

```bash
squeue -u $USER
```

### 6.2 GPU 作业示例

如果你的程序需要 GPU，推荐参考下面这种写法。这个思路更接近我们目前实际在用的脚本，例如：

`/mimer/NOBACKUP/groups/naiss2026-4-521/PulseBat/gpu_combo_all/run_gpu_combo_all_lfp3_fai_irrev_bad_crate_rerun.sbatch`

保存为 `run_gpu.sbatch`：

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

补充说明：

- CPU-only 任务建议显式加 `#SBATCH -C NOGPU`；这是 C3SE 官方文档中对 Alvis 无 GPU 节点的推荐用法
- 这里的 `module load PyTorch/...` 不是说你的代码一定要用 PyTorch，而是因为这个公共模块已经把一套兼容的 GPU/CUDA 运行时准备好了，后面再叠加我们自己的 `bamm_venv_gpu`
- 如果你的 GPU 环境是按别的模块建出来的，就把这里换成和你环境匹配的模块
- 常见 GPU 类型包括 `A100`、`A100fat`、`A40`、`V100`、`T4`
- 如果你需要指定卡型，可以写成类似 `#SBATCH --gpus-per-node=A100:1` 或 `#SBATCH --gpus-per-node=T4:1`
- 当前可用的 GPU 卡型、硬件信息和实时队列状态，可以查看 C3SE 官方页面：`https://www.c3se.chalmers.se/about/Alvis/`
- 该页面的 `Queue` 区域会链接到实时状态页面；根据 C3SE 官方说明，这部分通常只能在 SUNET 网络内访问，校外一般需要先连接 VPN
- 在 Alvis 命令行里，也可以用 `jobinfo`、`squeue`、`sinfo` 查看排队和资源情况

## 7. VS Code 通过 SSH 连接 Alvis

如果你已经能在本地终端正常 SSH 登录 Alvis，那么接入 VS Code Remote-SSH 通常只差一步。

### 7.1 本地 SSH 配置

在你自己电脑上的 `~/.ssh/config` 中增加一段类似配置：

```sshconfig
Host alvis
    HostName <your-alvis-login-host>
    User <your-cid>
    IdentityFile ~/.ssh/alvis1_local
```

说明：

- `Host alvis` 是你自己起的别名
- `HostName` 请替换成 C3SE/Alvis 提供给你的实际登录地址
- `User` 替换成你的用户名
- `IdentityFile` 替换成你本地私钥路径

如果你本来已经可以在终端里这样登录：

```bash
ssh alvis
```

那么 VS Code 里直接复用这个别名即可。

### 7.2 VS Code 配置流程

1. 在本地电脑安装 VS Code
2. 安装扩展 `Remote - SSH`
3. 打开命令面板，执行 `Remote-SSH: Connect to Host...`
4. 选择刚才配置好的 `alvis`
5. 首次连接时等待 VS Code 在远端安装服务端组件
6. 连接成功后，就可以像打开本地工程一样打开 Alvis 上的目录

建议打开的目录：

- 代码目录
- 项目组空间中的工作目录
- 例如 `/mimer/NOBACKUP/groups/naiss2025-22-468`

提醒：

- `~/.ssh/config` 这类 SSH 配置必须放在你自己的本地电脑或个人家目录里
- 它不是放在项目组空间里的

## 8. 网页端 Jupyter Notebook / JupyterLab 环境挂载

在 Alvis 的网页端入口中，可以通过 `~/portal/jupyter/*.sh` 脚本让自己的环境出现在 Jupyter 选项里。

例如你现在已有一个脚本：

```bash
/cephyr/users/chunqiu/Alvis/portal/jupyter/py312_bamm_seaborn.sh
```

对于每个人来说，这个路径里的用户名要替换成自己的用户名。

一个典型脚本如下：

```bash
ml purge
ml Python/3.12.3-GCCcore-13.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/bamm_venv/bin/activate
jupyter lab --config="${CONFIG_FILE}" --FileCheckpoints.checkpoint_dir='/mimer/NOBACKUP/groups/naiss2025-22-468/.ipynb_uni'
```

这段脚本的作用是：

- 先清理模块环境
- 加载和虚拟环境匹配的 Python 模块
- 激活组空间里的 `bamm_venv`
- 启动网页端 JupyterLab
- 把 notebook 的 checkpoint 也放到组空间

### 8.1 新增自己的 Jupyter 入口

例如：

```bash
mkdir -p ~/portal/jupyter
cp ~/portal/jupyter/py312_bamm_seaborn.sh ~/portal/jupyter/py311_pybamm.sh
chmod +x ~/portal/jupyter/py311_pybamm.sh
```

然后把内容改成你自己的环境，例如：

```bash
ml purge
ml Python/3.11.3-GCCcore-12.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/envs/py311_pybamm/bin/activate
jupyter lab --config="${CONFIG_FILE}" --FileCheckpoints.checkpoint_dir='/mimer/NOBACKUP/groups/naiss2025-22-468/.ipynb_uni'
```

这样在网页端 Jupyter 的环境列表中，就更容易找到并选择你自己的入口。

## 9. 一个适合组内新同学的推荐流程

如果你是第一次使用 Alvis，建议按下面的顺序操作：

1. 先学会 SSH 登录和 VS Code Remote-SSH 连接
2. 先学会 `module purge` 和 `module load`
3. 先尝试公共环境是否够用
4. 如果项目确实依赖 PyBaMM 和特定版本库，再去组空间建 `venv`
5. 运行正式任务时，尽量用 `sbatch`
6. 如果要在网页端做 notebook，再到 `~/portal/jupyter/` 下增加自己的启动脚本

## 10. 常用命令速查

```bash
# 查看可用模块
module avail Python
module avail CUDA
module spider PyTorch

# 加载公共 Python
module purge
module load Python/3.12.3-GCCcore-13.3.0

# 创建自己的 PyBaMM 环境
cd /mimer/NOBACKUP/groups/naiss2025-22-468
module purge
module load Python/3.11.3-GCCcore-12.3.0
python -m venv envs/py311_pybamm
source envs/py311_pybamm/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install -r /path/to/requirements.txt

# 激活已有环境
module purge
module load Python/3.11.3-GCCcore-12.3.0
source /mimer/NOBACKUP/groups/naiss2025-22-468/envs/py311_pybamm/bin/activate

# 提交任务
sbatch run_cpu.sbatch
sbatch run_gpu.sbatch

# 查看任务
squeue -u $USER
```

## 11. 最后的建议

- 不确定时，先用公共环境，不要急着自己装
- 要装环境时，优先装到项目组空间
- 对 PyBaMM 这类依赖敏感的项目，优先固定 Python 版本和 `requirements.txt`
- 运行 `venv` 前，记得先加载创建它时对应的 Python 模块
- 正式任务尽量走 `sbatch`
- Jupyter 和 VS Code 只是入口，真正占空间的环境、数据、日志尽量放到大空间里
