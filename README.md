# 面向肺部 CT 异构病灶评估系统

## 1、作品功能
本作品面向肺部CT影像辅助分析需求，设计并实现了一套基于AMD Kria KV260 FPGA平台的异构病灶智能评估系统，用以辅助解决传统人工阅片工作量大、病灶定位与范围判断依赖医生经验、定量评估效率较低等问题。系统采用ARM处理器、DPU神经网络加速单元与自定义HLS硬件加速器协同架构，实现从医学影像输入、病灶自动分割到肺区约束下的定量分析及报告输出的完整处理流程。

系统具备以下核心功能：

①**病灶智能分割**，利用轻量化deep_cap U-Net网络识别CT图像中的疑似病灶；

②**异构硬件加速**，采用DPU完成神经网络推理，并通过自定义HLS加速器实现病灶阈值判断、掩膜生成、肺区交集；

③**定量评估**，计算肺内病灶影像受累面积比例，提供可复核的量化依据；

④**交互式分析**，通过Web平台完成图像上传、任务管理、分割结果可视化及HTML、JSON报告导出。

## 2、整体架构
本系统采用基于AMD Kria KV260开发平台的PS/PL异构计算架构，构建面向肺部CT病灶智能分割与定量评估的软硬件一体化处理系统。系统主要由PC端Web交互模块、PS处理与调度模块、PL异构计算模块以及DDR存储与AXI互连模块组成。

系统设计的核心思想是依据不同计算任务的特点进行软硬件功能划分：将具有复杂控制逻辑、较强灵活性和较低并行度的任务交由ARM处理器执行；将计算密集型深度学习网络映射至DPU神经网络加速单元；将结构固定、数据并行度高的病灶区域后处理映射至自定义HLS加速器。

各功能模块通过统一的数据组织方式、共享存储空间和硬件控制接口建立联系，实现计算资源的协同利用。
<img width="431" height="182" alt="image" src="https://github.com/user-attachments/assets/2b5ef17e-0f55-430d-ac77-eaa1dd53d097" />

## 3、模块

**PC端Web交互模块** 是系统的人机交互入口，负责接收用户上传的CT图像、创建分析任务、查询处理状态，以及展示病灶分割图像、量化指标和分析报告。PC端通过以太网与KV260板端服务通信，不直接参与底层FPGA计算。

**PS处理与调度模块**是系统的控制中心。ARM Cortex-A53运行Linux系统，负责图像解码、输入数据预处理、肺区估计、硬件运行时初始化、任务调度、内存管理和结果整合。PS端分别通过VART和XRT管理DPU推理任务及HLS计算任务。

**PL异构计算模块**是系统的主要计算部分，由DPU和自定义HLS加速器组成。其中，DPU部署经过INT8量化的deep_cap U-Net网络，完成疑似病灶自动分割；HLS加速器根据DPU输出的病灶概率数据及肺区掩膜，执行阈值处理、掩膜生成、肺区交集及病灶区域统计。

**存储与互连模块**负责各计算单元之间的数据共享和通信。系统采用DDR保存模型参数、输入图像、中间张量、病灶概率图、肺区掩膜和统计结果，并通过AXI接口建立PS、DPU与HLS之间的控制及数据访问路径。

## 目录

```text
/A-System-for-Evaluating-Heterogeneous-Lesions-in-Lung-CT-Images/kv260-lung-ct.rar
|--src/       HLS、PS 与训练源码
|--sim/       原生功能测试、合成闭环及本次结果
|--build/     Vitis 2022.1 分阶段构建、实现报告采集
|--board/     板端启动脚本、配置与历史实测归档
|--data/      合成测试数据、参考结果和模型获取约定
|--skill/     可移植协作技能包
|--report/    设计、复现边界、证据和协作记录
```

## 训练与量化

Kaggle archive 入口：`src/training/covid19_lesion_kaggle_kv260_archive.ipynb`。较新的 deep_cap 训练/导出材料位于 `src/training/structure_optimization`（以该目录文件为准）。两种模型的结构、预处理和阈值协议不同，不可互换权重或编译产物。模型、数据和平台依赖见 [data/README.md](data/README.md)。

旧 archive 模型的主机脚本：`src/training/rebuild_for_vitis25.py`、`src/ps/quantize_vitis25.py`；deep_cap 使用 `src/ps/deployment/deep_cap/model_tools.py`。必须先通过 FP32 数值对齐，再量化并根据实际指纹编译。

## Vitis 构建

开发主机使用 Vivado/Vitis/Vitis HLS 2022.1；模型工具链使用 Vitis AI 2.5。

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

在实际工具链主机执行，各步骤失败即停止。源码或配置改变时使用新的 `--build` 目录。HLS源码的低资源分支、时序和资源要求见 [构建说明](build/README.md)。

## 板端运行

正确加载匹配固件，再进入具有 VART/XIR/XRT 的板端容器。

```bash
bash board/scripts/run_ps.sh --image /path/CT.png \
  --model /path/matching_model.xmodel --case-id case001 \
  --output /path/new_result
```

使用联合固件时显式增加 `--backend xrt-native --xclbin /path/metadata.xclbin --firmware-uuid ACTUAL_UUID`。读取 [board/README.md](board/README.md)，不要将 SmartCam 原固件当成包含自定义HLS的固件。模型需符合 deep_cap 的编译图检查；旧archive模型可用 `run_dpu_slice.py` 独立入口。
