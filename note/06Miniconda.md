# 一、Miniconda 安装与基础配置

## 1.1 文档信息

- **文档名称**：Windows Miniconda 安装与 Python 机器学习环境配置指南
- **操作系统**：Windows
- **适用范围**：GEE、Python、机器学习、遥感数据处理环境
- **Miniconda 安装目录**：`D:\Software\sf1\miniconda`
- **Conda 包缓存目录**：`D:\Software\sf1\miniconda\pkgs`
- **Conda 虚拟环境目录**：`D:\Software\sf1\miniconda\envs`

## 1.2 安装 Miniconda

建议将 Miniconda 安装到非系统盘，例如：

```text
D:\Software\sf1\miniconda
```

安装完成后，可以通过 PowerShell 检查 Conda：

```powershell
conda --version
```

如果能够正常显示版本号，说明 Miniconda 已经安装成功。

当前使用的 Conda 版本为：

```text
conda 26.7.1
```

## 1.3 检查 Conda 基础信息

在 Windows PowerShell 中执行：

```powershell
conda info
```

重点检查：

```text
base environment
```

应该指向：

```text
D:\Software\sf1\miniconda
```

同时检查 Conda 是否具有对该目录的写入权限。

---

# 二、配置 Conda 包缓存与虚拟环境目录

## 2.1 为什么要单独配置 `pkgs` 和 `envs`

Conda 中的：

```text
pkgs
```

和：

```text
envs
```

用途不同。

`pkgs` 是 Conda 的**软件包缓存目录**，用于保存下载后的 Conda 软件包及其解压缓存。

`envs` 是 Conda 的**虚拟环境目录**，实际创建的 Python 环境会放在这里。

例如：

```text
D:\Software\sf1\miniconda
├── pkgs
│   └── Conda 软件包缓存
│
└── envs
    ├── gee_py
    └── 其他虚拟环境
```

安装 `numpy` 时，并不是简单地把下载文件直接放进：

```text
envs\gee_py
```

Conda 通常会先将软件包下载并缓存到：

```text
pkgs
```

然后再将需要的文件安装到：

```text
envs\gee_py
```

这样可以让多个虚拟环境复用已经下载的软件包，从而减少重复下载。

例如 `gee_py`、`ml_env`、`test_env` 三个环境都需要 `numpy` 时，Conda 会直接使用 `pkgs` 里已缓存的软件包，不需要每个环境各下载一次完整安装包。也就是说，`pkgs` 相当于 Conda 的本地软件包仓库，与"当前项目用哪个环境"无关。

## 2.2 将 Conda 包缓存设置到 D 盘

执行：

```powershell
conda config --add pkgs_dirs D:\Software\sf1\miniconda\pkgs
```

检查：

```powershell
conda config --show pkgs_dirs
```

理想结果：

```text
pkgs_dirs:
  - D:\Software\sf1\miniconda\pkgs
```

如果存在不需要的 C 盘缓存路径，可以在确认后删除对应配置。

最终希望只保留：

```text
D:\Software\sf1\miniconda\pkgs
```

## 2.3 将 Conda 虚拟环境设置到 D 盘

执行：

```powershell
conda config --add envs_dirs D:\Software\sf1\miniconda\envs
```

检查：

```powershell
conda config --show envs_dirs
```

主要使用：

```text
D:\Software\sf1\miniconda\envs
```

之后创建：

```text
gee_py
```

等虚拟环境时，环境实际位置会是：

```text
D:\Software\sf1\miniconda\envs\gee_py
```

## 2.4 检查用户级 `.condarc`

Windows 用户级 Conda 配置文件通常位于：

```text
C:\Users\34936\.condarc
```

当前配置应包含类似内容：

```yaml
pkgs_dirs:
  - D:\Software\sf1\miniconda\pkgs

envs_dirs:
  - D:\Software\sf1\miniconda\envs
```

检查配置来源：

```powershell
conda config --show-sources
```

需要注意：

```text
C:\Users\34936\.conda
```

以及：

```text
C:\Users\34936\AppData\Local\conda\conda
```

可能仍然存在，但这并不代表 Conda 正在将完整的环境或软件包安装到 C 盘。

如果其中只有少量：

```text
aau_token
aau_token_host
environments.txt
tos
Cache
notices
```

等配置、认证和通知文件，通常不需要删除。

---

# 三、配置国内 Conda 镜像

## 3.1 使用清华大学 TUNA 镜像

如果网络环境位于中国大陆，可以将 Conda 软件源配置为清华大学 TUNA 镜像，以提高 Conda 软件包下载速度。

编辑：

```text
C:\Users\34936\.condarc
```

可以使用以下配置：

```yaml
channels:
  - defaults
  - conda-forge

default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2

custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud

channel_priority: strict
show_channel_urls: true

pkgs_dirs:
  - D:\Software\sf1\miniconda\pkgs

envs_dirs:
  - D:\Software\sf1\miniconda\envs
```

## 3.2 `custom_channels` 的作用

如果使用：

```yaml
channels:
  - conda-forge
```

仅仅添加 `conda-forge` 并不能保证 Conda 会使用清华镜像。

因此需要配置：

```yaml
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

这样：

```text
conda-forge
```

才会映射到清华 TUNA 的对应镜像。

## 3.3 查看实际配置

执行：

```powershell
conda config --show-sources
```

然后：

```powershell
conda config --show channels
```

再检查：

```powershell
conda config --show default_channels
```

同时检查：

```powershell
conda config --show pkgs_dirs
```

以及：

```powershell
conda config --show envs_dirs
```

## 3.4 测试 Conda 软件源

可以执行：

```powershell
conda search numpy
```

如果能够正常查询到 `numpy` 软件包及其版本信息，说明 Conda 软件源基本正常。

---

# 四、测试 Conda 环境是否能够正确创建到 D 盘

## 4.1 创建测试环境

为了确认 `envs_dirs` 配置有效，可以创建一个临时环境：

```powershell
conda create -n test_conda python=3.11
```

创建过程中如果出现确认提示，输入：

```text
a
```

或根据当前提示选择接受。

## 4.2 激活测试环境

```powershell
conda activate test_conda
```

检查 Python：

```powershell
python --version
```

检查 Python 实际位置：

```powershell
where python
```

正常情况下，第一个路径应该是：

```text
D:\Software\sf1\miniconda\envs\test_conda\python.exe
```

例如：

```text
D:\Software\sf1\miniconda\envs\test_conda\python.exe
D:\Software\sf1\miniconda\python.exe
D:\Program Files\Python\python.exe
C:\Users\34936\AppData\Local\Microsoft\WindowsApps\python.exe
```

其中最重要的是：

```text
D:\Software\sf1\miniconda\envs\test_conda\python.exe
```

位于第一位。

这说明当前激活的 Conda 环境优先级正常。对任何环境（包括后面的 `gee_py`）都用同样的三步核对：`python --version` 看版本、`where python` 看实际路径、`conda env list` 看环境位置。

## 4.3 检查环境位置

执行：

```powershell
conda env list
```

应该能够看到：

```text
test_conda    D:\Software\sf1\miniconda\envs\test_conda
```

这证明虚拟环境已经成功创建在 D 盘。

## 4.4 删除测试环境

测试完成后，不需要长期保留 `test_conda`。

先退出：

```powershell
conda deactivate
```

然后删除：

```powershell
conda remove -n test_conda --all
```

确认时输入：

```text
y
```

---

# 五、Conda 安装过程中的 `conda-pypi` WARNING

## 5.1 WARNING 的含义

在使用 Conda 26.7.1 时，可能出现：

```text
WARNING conda.conda_pypi.main:notify_externally_managed_future(156):
  Did you know? You can install many PyPI packages with conda
  using the conda-pypi beta. Get started:
    https://docs.conda.io/projects/conda/en/stable/new-features.html
```

这个信息不是安装错误。

它主要是在介绍 Conda 新增的：

```text
conda-pypi beta
```

功能。

## 5.2 是否需要处理

目前不需要。

这个提示不会影响：

- Conda 创建虚拟环境
- Python 安装
- `numpy` 安装
- `pandas` 安装
- `scikit-learn` 安装
- GEE Python API
- PyCharm
- D 盘 `pkgs`
- D 盘 `envs`

因此看到该 WARNING 时，可以继续正常操作。

---

# 六、创建正式的 GEE Python 环境

## 6.1 为什么使用 Python 3.9

之前的 GEE 项目环境使用的是：

```text
Python 3.9.23
```

为了保持项目兼容性，正式的 GEE 环境继续使用 Python 3.9，而不是直接使用 Miniconda Base 环境中的 Python 3.14。

这样可以减少：

- Python 版本兼容问题
- GEE 依赖冲突
- GIS 库版本冲突
- 机器学习库兼容问题

## 6.2 创建 `gee_py`

首先确保不在测试环境中：

```powershell
conda deactivate
```

然后创建：

```powershell
conda create -n gee_py python=3.9
```

创建完成后激活：

```powershell
conda activate gee_py
```

## 6.3 检查环境

检查方法与第四章相同，只是把对象换成 `gee_py`：

1. `python --version` 应显示 `Python 3.9.x`。
2. `where python` 的第一个路径应该是 `D:\Software\sf1\miniconda\envs\gee_py\python.exe`，说明当前使用的是 `gee_py` 环境中的 Python。
3. `conda env list` 应能看到：

```text
base       D:\Software\sf1\miniconda
gee_py     D:\Software\sf1\miniconda\envs\gee_py
```

---

# 七、GEE 与机器学习环境的软件包规划

## 7.1 核心 GEE 软件包

正式环境主要需要：

```text
earthengine-api
geemap
```

其中：

- `earthengine-api`：Google Earth Engine Python API。
- `geemap`：用于 GEE Python 环境中的交互式地图和遥感数据处理。

## 7.2 机器学习软件包

洪涝检测和随机森林实验需要：

```text
numpy
pandas
scikit-learn
```

其中：

- `numpy`：数值计算。
- `pandas`：表格数据处理。
- `scikit-learn`：随机森林、分类评价等机器学习功能。

## 7.3 常用遥感与 GIS 软件包

根据后续项目需要，可以安装：

```text
matplotlib
geopandas
rasterio
```

用途分别为：

- `matplotlib`：绘图和结果可视化。
- `geopandas`：矢量数据处理。
- `rasterio`：GeoTIFF 等栅格数据读写。

## 7.4 不建议一次安装所有软件包

建议按照：

```text
创建环境
    ↓
安装核心 Python 工具
    ↓
安装 GEE
    ↓
测试 GEE
    ↓
安装机器学习库
    ↓
安装 GIS / 遥感库
    ↓
配置 PyCharm
```

逐步进行。

这样如果出现依赖冲突，可以更容易确定具体是哪一个软件包导致的问题。

---

# 八、PyPI 与 pip 镜像配置

## 8.1 Conda 镜像与 pip 镜像的区别

Conda 和 pip 是两套不同的软件包管理方式。

Conda 主要使用：

```text
Conda channel
```

例如：

```text
defaults
conda-forge
```

而 pip 使用：

```text
PyPI
```

因此，配置 Conda TUNA 镜像后，并不意味着 pip 自动使用 TUNA 镜像。

## 8.2 配置清华 PyPI 镜像

如果需要提高 pip 安装速度，可以执行：

```powershell
python -m pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

检查：

```powershell
python -m pip config list
```

如果显示：

```text
global.index-url='https://pypi.tuna.tsinghua.edu.cn/simple'
```

说明 pip 镜像配置成功。

---

# 九、最终目录结构

## 9.1 Miniconda 主目录

最终整体结构可以是：

```text
D:\Software\sf1\miniconda
├── condabin
├── envs
│   └── gee_py
│       ├── python.exe
│       ├── Lib
│       ├── Scripts
│       └── ...
├── pkgs
│   ├── numpy
│   ├── pandas
│   ├── scikit-learn
│   └── ...
├── Library
├── Scripts
└── python.exe
```

## 9.2 用户配置文件

Conda 用户级配置：

```text
C:\Users\34936\.condarc
```

主要负责：

```text
Conda 软件源
Conda 包缓存目录
Conda 虚拟环境目录
channel 优先级
```

除 `.condarc` 之外，Conda 还会在 `C:\Users\34936\.conda` 与 `C:\Users\34936\AppData\Local\conda\conda` 保留少量配置、认证、环境注册与通知文件。它们不代表软件包或虚拟环境被装到了 C 盘，通常不需要删除，具体判断见第二章第 4 节。

## 9.3 项目环境

最终 GEE 项目的 Python 环境：

```text
D:\Software\sf1\miniconda\envs\gee_py
```

PyCharm 后续应该将这个环境中的 `python.exe` 设置为项目解释器：

```text
D:\Software\sf1\miniconda\envs\gee_py\python.exe
```

---

# 十、最终检查清单

全部配置完成后，按顺序执行下面的命令核对结果。

| 检查项 | 命令 | 预期结果 |
| --- | --- | --- |
| Miniconda | `conda --version` | 正常显示版本号 |
| Base 环境 | `conda info` | `base environment` 指向 `D:\Software\sf1\miniconda` |
| 包缓存 | `conda config --show pkgs_dirs` | 主要使用 `D:\Software\sf1\miniconda\pkgs` |
| 虚拟环境 | `conda config --show envs_dirs` | 主要使用 `D:\Software\sf1\miniconda\envs` |
| 软件源 | `conda config --show channels`、`conda config --show default_channels` | 已配置清华 TUNA 镜像 |
| GEE 环境 | `conda activate gee_py` 后执行 `python --version`、`where python` | Python 3.9，首个路径来自 `D:\Software\sf1\miniconda\envs\gee_py\` |
| 环境列表 | `conda env list` | 能看到 `base` 与 `gee_py`，且路径都在 `D:\Software\sf1\miniconda` 下 |

各项的配置方法分别见第一、二、三、六章。

---

# 十一、推荐的最终环境管理方式

## 11.1 Base 环境

`base` 环境主要用于：

- Conda 本身管理
- 创建和删除虚拟环境
- Conda 配置

不建议把大量项目依赖全部安装到 `base`。

## 11.2 `gee_py` 环境

你的 GEE、遥感和机器学习项目统一使用 `gee_py`：

```powershell
conda activate gee_py    # 进入环境
conda deactivate         # 退出环境
```

## 11.3 PyCharm

PyCharm 项目解释器设置为：

```text
D:\Software\sf1\miniconda\envs\gee_py\python.exe
```

调用链如下：

```text
PyCharm
   ↓
gee_py
   ↓
Python 3.9
   ↓
earthengine-api
geemap
numpy
pandas
scikit-learn
GIS / 遥感库
```

整个武汉洪涝遥感项目就可以保持在独立的 Python 环境中。

---

# 十二、最终配置总结

## 12.1 核心路径

| 项目 | 路径 |
|---|---|
| Miniconda | `D:\Software\sf1\miniconda` |
| Conda 包缓存 | `D:\Software\sf1\miniconda\pkgs` |
| Conda 虚拟环境 | `D:\Software\sf1\miniconda\envs` |
| GEE Python 环境 | `D:\Software\sf1\miniconda\envs\gee_py` |
| Conda 用户配置 | `C:\Users\34936\.condarc` |

## 12.2 核心原则

- **Miniconda 安装到 D 盘。**
- **Conda 包缓存放到 D 盘。**
- **Conda 虚拟环境放到 D 盘。**
- **C 盘保留少量 Conda 用户配置和系统级缓存即可，不要为了彻底清空 C 盘而删除 Conda 配置文件。**
- **GEE 项目使用独立的 `gee_py` 环境，`base` 环境不作为主要项目开发环境。**
- **Python 版本优先保持 Python 3.9，以兼容现有 GEE 项目。**
- **Conda 与 pip 的软件源需要分别配置。**
- **`conda-pypi` WARNING 不属于安装错误，可以正常忽略。**
- **在正式安装大量依赖前，先按第十章清单确认 Conda、镜像、`pkgs` 和 `envs` 配置正确。**

---
