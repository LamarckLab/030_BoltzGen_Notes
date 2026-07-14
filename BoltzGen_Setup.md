## Lamarck &nbsp; &nbsp; &nbsp; 2026-07-13
#### 该文档用于部署 BoltzGen
---

## 01  克隆官方源码仓库
> **236 机子路径 /data/lmk/boltzgen**
```bash
cd /data/lmk
git clone https://github.com/HannesStark/boltzgen.git
```

## 02  创建 conda 环境
> 官方要求 Python >= 3.11；依赖里 `numpy==2.0.2`、`numba==0.61.0` 锁死版本，必须开全新环境
```bash
conda create -n lmk_BoltzGen python=3.12 -y
conda activate lmk_BoltzGen
```

## 03  可编辑安装 BoltzGen
> `-e` 让 env 直接链接到源码目录，改代码或 git pull 后无需重装
```bash
cd /data/lmk/boltzgen
python -m pip install -e .
```

## 04  修复 torch 版本
> `pyproject.toml` 只写了 `torch>=2.4.1`，不锁 CUDA，pip 会拉最新的 `2.12.1+cu130`（CUDA 13）；236 驱动只到 CUDA 12.8 → `torch.cuda.is_available()` 为 False。降版本
```bash
python -m pip install torch==2.11.0 --index-url https://download.pytorch.org/whl/cu128
```
> 必须用 `--index-url`（独占源）。用 `--extra-index-url` 会因为 `+cu130 > +cu128` 又把 CUDA 13 那个版本选回来

## 05  修复 SOCKS 代理缺失的 socksio
> 236 上设了 SOCKS 代理，huggingface_hub 底层的 httpx 走 SOCKS 需要 `socksio`，否则下载权重时会直接报 `ImportError: Using SOCKS proxy, but the 'socksio' package is not installed`
```bash
python -m pip install "httpx[socks]"
```

## 06  下载模型权重
> 共 6 个文件约 10GB：2 个 design ckpt + inverse-fold + folding（Boltz-2）+ affinity + moldir；用 `--cache` 指到数据盘，避免落进家目录
```bash
boltzgen download all --cache /data/lmk/boltzgen_downloads
```

## 07  验证
> torch 认到 GPU（返回 True）+ 在 GPU 上跑一个算子 + cuequivariance 能 import（确认降 torch 版本后 ABI 没崩）
```bash
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
python -c "import torch; print((torch.randn(3, device='cuda') + 1).sum().item())"
python -c "import cuequivariance_torch as cet; print(cet.__version__)"
```

## 08  跑通官方示例
> 端到端验收：设计一条 80–140 aa 的蛋白去结合 1G13 的 A 链，完整走一遍 design → inverse_folding → folding → design_folding → analysis → filtering 六步流水线
```bash
cd /data/lmk/boltzgen
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=2 \
boltzgen run example/vanilla_protein/1g13prot.yaml \
  --output /data/lmk/boltzgen_outputs/test_run \
  --protocol protein-anything \
  --num_designs 10 \
  --budget 2 \
  --cache /data/lmk/boltzgen_downloads
```
> 首跑会现场编译 cuequivariance / triton 算子，卡住几分钟无输出是正常现象；`final_ranked_designs/` 里生成结果即部署成功

## 09  BoltzGen 权重与输入输出目录
**236 机子路径**
> 权重缓存：/data/lmk/boltzgen_downloads  &nbsp;（`boltzgen download all` 下载，约 10GB）
> 输入目录：/data/lmk/boltzgen_inputs  &nbsp;（设计规范 yaml + 靶点 cif/pdb）
> 输出目录：/data/lmk/boltzgen_outputs  &nbsp;（设计结果 cif + 指标 csv + 结果图 pdf）

##### [BoltzGen 官方仓库](https://github.com/HannesStark/boltzgen)
