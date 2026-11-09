# 面向肺部 CT 异构病灶评估系统

基于 KV260 的单切片病灶分割、传统肺区估计、面积受累分级与报告系统。DPU 执行 U-Net，PS 或自定义 HLS 执行肺区/后处理，输出灰度 CT＋红色病灶叠加图和 HTML/JSON 报告。

## 目录

```text
/A-System-for-Evaluating-Heterogeneous-Lesions-in-Lung-CT-Images/kv260-lung-ct.rar
src/       HLS、PS 与训练源码
sim/       原生功能测试、合成闭环及本次结果
build/     Vitis 2022.1 分阶段构建、实现报告采集
board/     板端启动脚本、配置与历史实测归档
data/      合成测试数据、参考结果和模型获取约定
skill/     可移植协作技能包
report/    设计、复现边界、证据和协作记录
```

## 训练与量化

Kaggle archive 入口：`src/training/covid19_lesion_kaggle_kv260_archive.ipynb`。较新的 deep_cap 训练/导出材料位于 `src/training/structure_optimization`（以该目录文件为准）。两种模型的结构、预处理和阈值协议不同，不可互换权重或编译产物。模型、数据和平台依赖见 [data/README.md](data/README.md)。

旧 archive 模型的主机脚本：`src/training/rebuild_for_vitis25.py`、`src/ps/quantize_vitis25.py`；deep_cap 使用 `src/ps/deployment/deep_cap/model_tools.py`。必须先通过 FP32 数值对齐，再量化并根据实际指纹编译。

## Vitis 构建

开发主机使用 Vivado/Vitis/Vitis HLS 2022.1；模型工具链使用 Vitis AI 2.5。板端历史基线为 Ubuntu 22.04、VART 2.5、XRT 2.13、B3136，指纹 `0x101000016010406`。这不是自动检测到的当前板卡状态。

```bash
cp build/vitis_20221/config.example.json build/vitis_20221/config.json
# 编辑实际平台.xpfm、匹配DPU.xo、时钟ID和AXI端口；示例值不是通用约束。
python build/vitis_20221/build.py check --config build/vitis_20221/config.json
python build/vitis_20221/build.py csim --config build/vitis_20221/config.json
python build/vitis_20221/build.py synth --config build/vitis_20221/config.json
python build/vitis_20221/build.py cosim --config build/vitis_20221/config.json
python build/vitis_20221/build.py export --config build/vitis_20221/config.json
python build/vitis_20221/build.py link --config build/vitis_20221/config.json
```

在实际工具链主机执行，各步骤失败即停止。源码或配置改变时使用新的 `--build` 目录。HLS源码的低资源分支、时序和资源要求见 [构建说明](build/README.md)。无原始平台/DPU IP时只能复现软件与C++功能，不能从本仓库独立生成最终固件。

## 板端运行

先在宿主正确加载匹配固件，再进入具有 VART/XIR/XRT 的板端容器。脚本不自动卸载、重载固件。

```bash
bash board/scripts/run_ps.sh --image /path/CT.png \
  --model /path/matching_model.xmodel --case-id case001 \
  --output /path/new_result
```

默认后处理为 NumPy；使用联合固件时显式增加 `--backend xrt-native --xclbin /path/metadata.xclbin --firmware-uuid ACTUAL_UUID`。读取 [board/README.md](board/README.md)，不要将 SmartCam 原固件当成包含自定义HLS的固件。模型需符合 deep_cap 的编译图检查；旧archive模型可用 `run_dpu_slice.py` 独立入口。

## 实现状态

有历史 DPU 推理、18组后处理HLS板端对照及2份保存整机任务审计材料；本次交付只重新运行本地功能测试。完整 routed 资源/时序/CDC/DRC 报告、三内核肺区HLS联合固件验收、公平ARM/HLS对照尚未全部取得。不得把历史摘要或pragma目标当作当前源码的综合实现结论。

分级只描述单切片估计面积受累。详见 [设计报告](report/design.md)、[证据状态](report/evidence_status.md) 和 [大模型协作记录](report/llm_collaboration.md)。

厂商 DPU、平台与工具依适用许可取得。
