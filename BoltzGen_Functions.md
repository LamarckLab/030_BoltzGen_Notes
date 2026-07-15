## Lamarck &nbsp; &nbsp; &nbsp; 2026-07-14
#### 该文档用于记录 server 上跑 BoltzGen 的各种命令
---

*236 机子的环境*
```bash
conda activate lmk_BoltzGen
```

*输入输出路径*
```bash
权重缓存:   /data/lmk/boltzgen_downloads          # 下载权重，约 10GB
输入目录:   /data/lmk/boltzgen_inputs             # 设计规范 yaml + 靶点 cif/pdb（两者必须同目录）
输出目录:   /data/lmk/boltzgen_outputs/<yaml名>   # 设计结果 cif + 指标 csv + 结果图 pdf
```

*GPU 选择*
```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=2
```

---

> **00 校验设计规范**

写完任何 yaml 都建议先 check 再 run。它能做：① 校验语法与引用（链 ID 是否存在、残基范围是否越界）；② 给 `--output` 时，写一个上色 cif，供肉眼确认设计意图。
```bash
boltzgen check /data/lmk/boltzgen_inputs/1g13prot.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot \
  --cache /data/lmk/boltzgen_downloads
```
> `--output` 可省略（省略则只校验、不出图）
> `--cache` 指向权重目录，否则会重新下载 `mols.zip`
> 输出子目录用 yaml 的文件名

check 的真正价值不只是"格式检查"——它最重要的是能抓**意图错误**：yaml 完全合法、run 也不报错，但含义定义错误（典型是结合位点编号写错、指到了另一块表面）。这类错误打开染色之后的 cif 看一眼才能发现，check 让你在烧几小时 GPU 之前就发现，打开上色 cif 只会看到靶点那条链，靶点残基会被标红，其余为蓝色

> **01 设计蛋白 binder**

最基础用法：设计一条指定长度范围的蛋白链去结合靶点，不限定结合位点

设计规范 `1g13prot.yaml`
```yaml
entities:
  - protein:
      id: C
      sequence: 80..140      # 设计一条 80–140 aa 的链，长度每次随机采样
  - file:
      path: 1g13.cif
      include:
        - chain:
            id: A            # cif 里有 A/B/C 三条链，只取 A 链当靶点
```
运行
```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=2 \
boltzgen run /data/lmk/boltzgen_inputs/1g13prot.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot \
  --protocol protein-anything \
  --num_designs 25 \
  --budget 5 \
  --cache /data/lmk/boltzgen_downloads
```
> `--protocol protein-anything` 用蛋白结合蛋白
> `--num_designs` 候选数（真实任务 10,000–60,000）
> `--budget` 最终精选数
> 最终结果在 `final_ranked_designs/`
> ⚠️ 输出 cif 里的链名会被重排，和 yaml 中定义的不一样，记得检查

**六步流水线**：design（扩散模型生成骨架，只有主链、没有序列）→ inverse_folding（给骨架设计氨基酸序列）→ folding（用 Boltz-2 把「设计序列 + 靶点」重新折叠，回测设计是否自洽）→ design_folding（binder 单独折叠，看离开靶点还成不成型）→ analysis（算指标）→ filtering（排序 + 去冗余，选出 budget 个）。本质是「生成 → 回测」闭环，等价于把 RFdiffusion → ProteinMPNN → AF2 那条老流水线打包进一个工具

> **02 指定结合位点**

在 01 的基础上加一个 `binding_types` 块，限定 binder 必须结合到靶点的指定残基

设计规范 `1g13prot_site.yaml`
```yaml
entities:
  - protein:
      id: C
      sequence: 80..140
  - file:
      path: 1g13.cif
      include:
        - chain:
            id: A
      binding_types:              # 只比 01 多这一个块
        - chain:
            id: A
            binding: 59..62       # 必须结合到 A 链 59-62 号残基
```
命令的形式和 01 完全一样，换成带 `binding_types` 的 yaml、输出到对应目录
```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=2 \
boltzgen run /data/lmk/boltzgen_inputs/1g13prot_site.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot_site \
  --protocol protein-anything \
  --num_designs 25 \
  --budget 5 \
  --cache /data/lmk/boltzgen_downloads
```
> `binding:` 结合位点，`not_binding:` 禁止结合区，可同时用；值支持逗号 `343,344,251`、范围 `57..71`、整链 `"all"`
> ⚠️ 编号是 mmCIF 的 `label_seq_id`（从 1 顺序数），**不是** PDB 中的 `auth_seq_id`，即和 ProteinMPNN `--position_list` 中编号方式类似
> ⚠️ 是**软引导**不是硬约束：模型被鼓励往那贴，多数命中但不保证每个都完美

> **03 裁剪靶点**

在 02 的基础上加一个 `include_proximity` 块，只保留结合位点附近的残基（在 yaml 中定义距离）、丢掉靶点链的其余部分。通常靶点动辄几百上千残基，但 binder 只结合一小块，裁剪能大幅降低 folding 所用算力与显存

设计规范 `1g13prot_crop.yaml`
```yaml
entities:
  - protein:
      id: C
      sequence: 80..140
  - file:
      path: 1g13.cif
      include:
        - chain:
            id: A
      include_proximity:          # 裁剪：只留靶点氨基酸附近残基
        - chain:
            id: A
            res_index: 59..62     # 参照区域（通常就用结合位点）
            radius: 15            # 保留距参照 <15 埃的残基（本例 162→30）
      binding_types:
        - chain:
            id: A
            binding: 59..62
```
命令的形式和 01 完全一样，换成带 `include_proximity` 的 yaml、输出到对应目录
```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=2 \
boltzgen run /data/lmk/boltzgen_inputs/1g13prot_crop.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot_crop \
  --protocol protein-anything \
  --num_designs 25 \
  --budget 5 \
  --cache /data/lmk/boltzgen_downloads
```
> 裁出来的是**空间口袋**不是序列区间：残基编号会**稀疏跳号**（如 `54..67, 130..139, 156..160`），序列上远、空间上近的残基被拼在一起
> ⚠️ 稀疏编号的 cif **ChimeraX 打不开**（IndexError），用 [molstar](https://molstar.org/viewer/) 看

##### [BoltzGen 官方仓库](https://github.com/HannesStark/boltzgen)
