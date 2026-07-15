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

写完任何 yaml 都建议先 check 再 run。它做三件事：① 校验语法与引用（链 ID 是否存在、残基范围是否越界、CCD 码是否有效）；② 报告结构文件里缺失（未解析）的残基/原子；③ 给了 `--output` 时写出一个上色 cif，供肉眼确认设计意图。纯 CPU、几秒出结果，不用选 GPU
```bash
boltzgen check /data/lmk/boltzgen_inputs/1g13prot.yaml \
  --output /data/lmk/boltzgen_outputs/1g13prot \
  --cache /data/lmk/boltzgen_downloads
```
> `--output` 可省略（省略则只校验、不出图）
> `--cache` 指向权重目录，否则会重新下载 `mols.zip`
> 输出子目录用 yaml 的文件名

check 的真正价值不只是"格式检查"——它最重要的是能抓**意图错误**：yaml 完全合法、run 也不报错，但含义定义错误（典型是结合位点编号写错、指到了另一块表面）。这类错误只有打开染色之后的 cif 看一眼才能发现，而 check 让你在烧几小时 GPU 之前就发现，打开上色 cif 只会看到靶点那条链，靶点残基会被标红，其余为蓝色

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
> `--protocol protein-anything` 用蛋白结合蛋白/多肽
> `--num_designs` 候选数（真实任务 10,000–60,000）
> `--budget` 最终精选数
> 最终结果在 `final_ranked_designs/`
> ⚠️ 输出 cif 里的链名会被重排，和 yaml 中定义的不一样，记得检查

**六步流水线**：design（扩散模型生成骨架，只有主链、没有序列）→ inverse_folding（给骨架设计氨基酸序列）→ folding（用 Boltz-2 把「设计序列 + 靶点」重新折叠，回测设计是否自洽）→ design_folding（binder 单独折叠，看离开靶点还成不成型）→ analysis（算指标）→ filtering（排序 + 去冗余，选出 budget 个）。本质是「生成 → 回测」闭环，等价于把 RFdiffusion → ProteinMPNN → AF2 那条老流水线打包进一个工具

##### [BoltzGen 官方仓库](https://github.com/HannesStark/boltzgen)
