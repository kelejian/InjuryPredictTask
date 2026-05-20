# InjuryPredictTask 说明

## 1. 项目定位

`InjuryPredictTask` 是面向论文写作与实验复现整理出的损伤预测独立项目，核心内容包括碰撞波形预测模块 `PulsePredict` 和含波形输入的损伤预测模块 `InjuryPredict`。

当前仓库不包含自适应乘员约束系统参数寻优代码，但损伤预测模型可作为该类系统的快速代理模型，用于在下游应用中评估不同碰撞工况和约束参数组合下的乘员损伤风险。

## 2. 项目结构

```text
InjuryPredictTask/
├─ common/                         # 共享特征顺序、路径、归一化、划分和损伤指标工具
├─ PulsePredict/                   # 碰撞波形预测模型及训练、测试代码
├─ InjuryPredict/                  # 损伤预测模型、processed .pt 数据准备和训练评估代码
├─ data/                           # 本地数据目录，通常不纳入版本管理
├─ prepare_data.py                 # 原始数据打包、划分和归一化配置生成入口
├─ run_pulse_injury_inference.py   # PulsePredict + InjuryPredict 联合推理入口
└─ ARS_Pipeline.md                 # 自适应约束系统应用背景说明
```

## 3. 共享数据接口

跨模块共享的数据接口由 [`common/settings.py`](./common/settings.py) 统一管理。默认目录结构如下：

```text
data/
├─ raw_packed/
│  └─ raw_data_packed.npz
├─ split_indices/
│  ├─ injury/
│  │  ├─ combined/
│  │  ├─ driver/
│  │  └─ passenger/
│  └─ pulse/
├─ processed/
│  └─ injury/
│     ├─ combined/
│     ├─ driver/
│     └─ passenger/
└─ normalization_config.json
```

当前约定：

- `pulse` 任务只保留一套完整划分。
- `injury` 任务保留 `combined / driver / passenger` 三套划分。
- `PulsePredict` 与 `InjuryPredict` 共用同一份 `normalization_config.json`。
- 默认训练和评估口径为 `combined`。

## 4. 数据准备

根目录脚本 [`prepare_data.py`](./prepare_data.py) 负责：

- 打包 `data/raw_packed/raw_data_packed.npz`。
- 生成 `data/split_indices/pulse/`。
- 生成 `data/split_indices/injury/{combined,driver,passenger}/`。
- 基于合并主副驾训练集生成共享 `data/normalization_config.json`。

运行方式参考：

```bash
conda activate pytorch
python -m prepare_data
```

默认读取 `data/source/distribution.csv` 和 `data/source/pulse_csv/`。如果原始数据仍放在其他位置，可继续通过 `--distribution` 和 `--pulse-dir` 覆盖。

## 5. 训练与评估入口

### 5.1 PulsePredict

```bash
python -m PulsePredict.train -c PulsePredict/config.json
python -m PulsePredict.test -r PulsePredict/saved/models/<experiment>/<run>/model_best.pth
```

训练结果与测试输出默认保存在 `PulsePredict/saved/`。

### 5.2 InjuryPredict

先将共享打包数据转换为 `InjuryPredict` 使用的 processed `.pt` 数据：

```bash
python -m InjuryPredict.Injurydata_prepare
```

默认使用 `PulsePredict/saved/models/HybridPulseCNN/0502_123240/` 下的 `model_best.pth` 和 `config.json`。如需切换波形预测模型，可传入 `--pulse-checkpoint` 和 `--pulse-config`。

再训练或评估损伤预测模型：

```bash
python -m InjuryPredict.train
python -m InjuryPredict.eval_model
```

评估入口默认使用 `InjuryPredict/runs/InjuryPredictModel_05021735/` 与 `best_val_loss.pth`。如需评估其他 run，可传入 `--run_dir` 和 `--weight_file`。

训练结果与评估输出默认保存在 `InjuryPredict/runs/`。

### 5.3 联合推理

`run_pulse_injury_inference.py` 用于从包含工况参数的 CSV 执行两阶段推理：先预测碰撞波形，再预测 `HIC15 / Dmax / Nij / AIS / MAIS`。

```bash
python run_pulse_injury_inference.py
```

默认输入为 `data/inference_inputs/case_parameters.csv`，默认模型分别来自 `PulsePredict/saved/models/HybridPulseCNN/0502_123240/` 和 `InjuryPredict/runs/InjuryPredictModel_05021735/`。输入 CSV 或模型 run 变化时，可用 `--input-csv`、`--pulse-checkpoint`、`--pulse-config`、`--injury-checkpoint` 和 `--injury-record` 覆盖。

## 6. 运行约定

- 建议在项目根目录运行命令。
- 建议统一使用 `python -m ...` 运行包内脚本。
- 如需安装依赖：

```bash
pip install -r requirements.txt
```

## 7. 进一步说明

- [PulsePredict/README.md](./PulsePredict/README.md)
- [InjuryPredict/README.md](./InjuryPredict/README.md)
- [ARS_Pipeline.md](./ARS_Pipeline.md)
