下面是为你整理的 Markdown 格式版本，可以直接复制使用或导入到任何支持 Markdown 的笔记软件中（如 Notion、Obsidian、GitHub README 等）：

```markdown
# SOT 测试基准选型清单

> **挑选原则**：场景覆盖广、缺陷标注细、有标准评测脚本（AUC/Precision/F1）、且与项目落地场景（无人机/ADAS/端侧）相关。

---

## 一、首选基准（覆盖大部分场景 + 缺陷属性）

### 1. OTB100 / OTB2015 — 缺陷属性覆盖最经典
- **数据规模**：100 段序列，每段标注 11 个挑战属性：光照变化(IV)、尺度变化(SV)、遮挡(OCC)、形变(DEF)、运动模糊(MB)、快速运动(FM)、面内旋转(IPR)、面外旋转(OPR)、出视野(OV)、背景杂乱(BC)、低分辨率(LR)。
- **评测指标**：Success Plot (AUC) + Precision Plot，可按属性分桶看弱项。
- **项目价值**：最直接对应 SOT 代码里的缺陷（遮挡/形变/快速运动/尺度变化都有专属逻辑）。先跑这个，能立刻定位 `getStatus` 状态机、几何异常检测、模板冻结在哪些属性上崩。
- **入口**：[benchmark-results (含 OTB 数据与脚本汇总)](https://github.com/foolwood/benchmark-results) | 论文: Wu et al., TPAMI 2015。

### 2. VOT Challenge (VOT2022/2023/2024) — 工业标准 + 长短时
- **数据规模**：提供 VOT-ST(短时) / VOT-LT(长时，含目标消失重现) / VOT-RGBD / VOT-RGBT / VOT-RTL。
- **关键指标**：EAO (Expected Average Overlap)、Accuracy、Robustness；长时还看 F1、PR、RE。
- **工具链**：自带官方工具链 `vot-toolkit`（Python/MATLAB），一键跑评、自动重初始化协议。
- **项目价值**：VOT-LT 直击代码最弱的“目标完全遮挡后重捕获”；EAO 对鲁棒性敏感，能暴露 `assert_occluded` / 扩张 2× 上限的不足。
- **入口**：[VOT Challenge 官网](http://www.votchallenge.net/) | [vot-toolkit](https://github.com/votchallenge/vot-toolkit)

### 3. LaSOT / LaSOT_ext — 大规模长时，最贴近实战
- **数据规模**：1400+ 段长序列（~3.5M 帧），含目标消失重现；类别丰富（动物/车辆/人/物）。
- **评测指标**：AUC、Precision、Norm Precision。
- **项目价值**：验证模板冻结在长时形变下的衰减，统计意义强（不像 OTB 只有 100 段）。
- **入口**：[LaSOT-Evaluation-Toolkit](https://github.com/HengLan/LaSOT-Evaluation-Toolkit) | [数据下载](http://vision.cs.stonybrook.edu/~lasot/)

### 4. GOT-10k — 类别泛化 (zero-shot)
- **数据规模**：1万+段、560+类别，仅训练集有标注，测试集需在线提交。
- **评测指标**：AO (Average Overlap)、SR (Success Rate)。
- **项目价值**：测对新类别的泛化——如果 Siamese 模板只在固定类别上训过，换类会掉多少，靠它量化。
- **入口**：[GOT-10k 官网](http://got-10k.aitestunion.com/) | 工具: `got10k-toolkit`

### 5. UAV123 / UAV123@10fps — 无人机视角（与项目高度相关）
- **数据规模**：123 段低空 UAV 视频，112K+ 帧，含小目标、视角剧变、快速运动。
- **项目价值**：项目落地含无人机/ADAS 场景，UAV123 是最对口的真实视角基准。
- **入口**：[UAV123 官网](https://uav123.org/)

---

## 二、补充基准（按特定缺陷挑）

| 想验证的缺陷 | 推荐基准 | 说明 |
| :--- | :--- | :--- |
| **长时消失+重捕获** | VOT-LT / OxUvA / LTB50 / LaSOT | 重捕获只有 2× 扩张，必测 |
| **超长序列** | TLP (Tracking Long Sequences) / OxUvA | 模板冻结长程衰减 |
| **形变/非刚体** | OTB100(DEF) / NFS | NFS 480FPS 高帧率，运动模糊多 |
| **颜色/光照** | TC128 (Template Color 128) | 彩色序列，IV/BC 属性 |
| **红外/抗无人机** | AntiUAV / VOT-RGBT | 双模态，含目标丢失 |
| **RGB-D 深度** | PTB / VOT-RGBD | 有深度辅助场景 |
| **端到端延迟/实时性** | 自建 + NFS (高帧率) | 端侧 FPS 评估 |

---

## 三、推荐测试矩阵（按优化阶段）

- **阶段一（修 bug + Tier-0）**:
  - `OTB100` 全跑 → 按属性分桶，定位最差 3 个属性
  - `VOT-LT` 子集 → 看 F1/重捕获率（暴露重捕获弱点）
- **阶段二（在线模板更新 + 重检测分支）**:
  - `LaSOT` test → AUC 对比前后（验证模板更新收益）
  - `VOT-LT` 全量 → 验证重检测分支提升
- **阶段三（追 SOTA）**:
  - `GOT-10k` test 提交（泛化）
  - `TrackingNet` test 提交
  - `UAV123`（落地场景）

---

## 四、评测工具/脚本（避免自己造轮子）

- **VOT Toolkit**：[https://github.com/votchallenge/vot-toolkit](https://github.com/votchallenge/vot-toolkit) — 协议最规范，建议直接对接。
- **pytracking 评测栈**：[https://github.com/ethereal-dev/pytracking](https://github.com/ethereal-dev/pytracking) — 内置 OTB/LaSOT/GOT-10k/TrackingNet/UAV123 的评测与结果可视化。
- **got10k-toolkit**：[https://github.com/got-10k/toolkit](https://github.com/got-10k/toolkit)
- **LaSOT Toolkit**：[https://github.com/HengLan/LaSOT-Evaluation-Toolkit](https://github.com/HengLan/LaSOT-Evaluation-Toolkit)
- **pysot（商汤）**：[https://github.com/STVIA/pysot](https://github.com/STVIA/pysot) — 含 OTB/VOT/LaSOT/TrackingNet 评测，文档全，最适合做对照基线。

> 💡 **推荐做法**：直接复用 `pysot` 或 `pytracking` 的评测代码（输入：跟踪结果 txt，每行一个 bbox 或 `frame_id+score`），把你的 SOT 输出转成它要求的格式即可，省去重写 AUC/EAO 计算。

---

## 五、对接到你的 SOT（落盘格式）

你的 `track()` 每帧输出 `TrackerInfo{box_info_, status_}`。评测要求一般是每帧一个 `x,y,w,h`（OTB/LaSOT）或 `x1,y1,x2,y2,score`（VOT）。注意：

- **LOST 帧的处理**：OTB 允许输出空（按 0 IoU 计）；VOT-LT 协议要求显式标“目标不在场”（score < 阈值或空），这正是你 `status_==LOST` 要映射的——你的状态机输出天然契合 VOT-LT 协议，建议优先用 VOT-LT 评。

---

## Sources
- [VOT Challenge 官方](http://www.votchallenge.net/)
- [VOT Toolkit](https://github.com/votchallenge/vot-toolkit)
- [LaSOT](http://vision.cs.stonybrook.edu/~lasot/)
- [GOT-10k](http://got-10k.aitestunion.com/)
- [TrackingNet](https://tracking-net.org/)
- [UAV123](https://uav123.org/)
- [pysot (STVIA)](https://github.com/STVIA/pysot)
- [pytracking](https://github.com/ethereal-dev/pytracking)
- [got10k-toolkit](https://github.com/got-10k/toolkit)
- [benchmark-results (OTB 汇总)](https://github.com/foolwood/benchmark-results)
```