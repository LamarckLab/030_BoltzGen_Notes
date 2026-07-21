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
boltzgen check /data/lmk/boltzgen_inputs/1g13prot.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot \
  --cache /data/lmk/boltzgen_downloads
```
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
boltzgen check /data/lmk/boltzgen_inputs/1g13prot_site.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot_site  \
  --cache /data/lmk/boltzgen_downloads
```
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

在 02 的基础上加一个 `include_proximity` 块，只保留结合位点附近的残基（在 yaml 中定义距离）、丢掉靶点链的其余部分。靶点通常动辄几百上千残基，但 binder 只结合一小块，裁剪能大幅降低 folding 所用算力与显存

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
boltzgen check /data/lmk/boltzgen_inputs/1g13prot_crop.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot_crop  \
  --cache /data/lmk/boltzgen_downloads
```
```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=2 \
boltzgen run /data/lmk/boltzgen_inputs/1g13prot_crop.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot_crop \
  --protocol protein-anything \
  --num_designs 25 \
  --budget 5 \
  --cache /data/lmk/boltzgen_downloads
```
> 裁出来的 cif 残基编号会**稀疏跳号**（如 `54..67, 130..139, 156..160`），稀疏编号的 cif 用 **ChimeraX 打不开**（IndexError），可以用网页端 [molstar](https://molstar.org/viewer/) 查看

> **04-1 结构可见性：隐藏结构**

用 `structure_groups` 控制「靶点结构给模型看多少」。`visibility` 不是可见度等级，而是**组号**：只有**同组、且组号 > 0** 的残基之间，相对位置才对模型可见。本例用 `0`：结构完全不给，模型只知序列

设计规范 `1g13prot_hide.yaml`（把靶点结构完全藏起来）
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
      structure_groups:           # 藏起靶点结构：模型只知序列、不知构象
        - group:
            id: A                 # 必须写具体链名
            visibility: 0
      binding_types:
        - chain:
            id: A
            binding: 59..62
```
命令的形式和 01 完全一样，换成带 `structure_groups` 的 yaml、输出到对应目录
```bash
boltzgen check /data/lmk/boltzgen_inputs/1g13prot_hide.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot_hide  \
  --cache /data/lmk/boltzgen_downloads
```
```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=2 \
boltzgen run /data/lmk/boltzgen_inputs/1g13prot_hide.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot_hide \
  --protocol protein-anything \
  --num_designs 25 \
  --budget 5 \
  --cache /data/lmk/boltzgen_downloads
```
> `0` 结构不给（与谁都不可见）；
> `1` 默认组，组内相对位置全可见（不写 `structure_groups` 即全 1）；
> `2`（或更大）另一个独立刚体组，组内可见、但与组 1 的相对位置不可见

> 被设计的残基 visibility 必须为 0（源码硬约束）

> **04-2 结构可见性：自动对接**

用 `visibility: 2` 把一部分结构划成**独立刚体组**：组内结构完整保留，但它与组 1 之间的相对位置不给模型——等于让模型**自己做 docking**。只有一条链时 `2` 和 `1` 无差别，必须有两块带结构的实体才显现

设计规范 `AB_docking.yaml`（1g13 的 A、B 两条链在晶体中紧密接触，界面在 A 的 34/36/99-102/117-120 位置）
```yaml
entities:
  - protein:
      id: G
      sequence: 20              # 凑数的设计肽（BoltzGen 必须有东西可设计）
  - file:
      path: 1g13.cif
      include:
        - chain:
            id: A
        - chain:
            id: B
      binding_types:
        - chain:
            id: A
            binding: 59..62     # 把设计肽拴在这，距 A-B 界面 30 埃，避免干扰
      structure_groups:
        - group:
            id: A
            visibility: 1       # A 为组 1
        - group:
            id: B
            visibility: 2       # B 独立成组 2 → A-B 相对位置不给，模型自己对接
```
```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=2 \
boltzgen run /data/lmk/boltzgen_inputs/AB_docking.yaml \
  --output /data/lmk/boltzgen_outputs/AB_docking \
  --protocol protein-anything \
  --num_designs 10 \
  --budget 2 \
  --cache /data/lmk/boltzgen_downloads
```

**visibility 2 是抗体设计的地基**：纳米抗体脚手架把框架区设成组 2、CDR 设成组 0，抗原默认组 1 —— 框架与抗原各自是刚体，但**两者怎么对接由模型决定**（相当于把 docking 融进生成过程），CDR 结构完全重新生成。官方 23 个用到 `visibility: 2` 的例子全部是抗体/纳米抗体脚手架，没有组 2 就做不了 de novo 抗体设计

##### [BoltzGen 官方仓库](https://github.com/HannesStark/boltzgen)
