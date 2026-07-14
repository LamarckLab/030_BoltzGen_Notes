## Lamarck &nbsp; &nbsp; &nbsp; 2026-07-14
#### 该文档用于记录 server 上跑 BoltzGen 的各种命令
---

*236 机子的环境*
```bash
conda activate lmk_BoltzGen
```

*输入输出路径*
```bash
权重缓存:   /data/lmk/boltzgen_downloads   # boltzgen download all 下载，约 10GB
输入目录:   /data/lmk/boltzgen_inputs      # 设计规范 yaml + 靶点 cif/pdb（两者必须同目录）
输出目录:   /data/lmk/boltzgen_outputs     # 设计结果 cif + 指标 csv + 结果图 pdf
```

*GPU 选择*
```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=2
```

---

> **00 校验设计规范**

写完任何 yaml 都建议先 check 再 run。校验语法与引用（链 ID 是否存在、残基范围是否越界、CCD 码是否有效），纯 CPU、几秒出结果
```bash
boltzgen check /data/lmk/boltzgen_inputs/1g13prot.yaml \
  --output /data/lmk/boltzgen_outputs \
  --cache /data/lmk/boltzgen_downloads
```
> `--output` 可省略（省略则只校验、不出图）
> `--cache` 指向权重目录，否则会重新下载 `mols.zip`

##### [BoltzGen 官方仓库](https://github.com/HannesStark/boltzgen)
