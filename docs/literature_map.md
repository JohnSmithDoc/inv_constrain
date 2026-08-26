# 纯惯导漂移抑制：领域地图与文献笔记

> 目标：不用任何绝对位置观测（无 GNSS/无视觉绝对定位），通过各种约束"榨干信息"，实现纯惯导定位漂移抑制。
> 调研日期：2026-08-26。数据来源：arXiv API / 论文全文（可验证）。

## 0. 参考起点

- **Bryson & Sukkarieh, "Vehicle Model Aided Inertial Navigation for a UAV using Low-cost Sensors"**（悉尼大学 ACFR，约 2004）
  - 飞行器动力学模型 FVM（六自由度刚体 + 气动系数 + 推力近似 T=Pmax·η/V）辅助低成本 IMU
  - 配置1：FVM 速度+姿态 与 INS 之差做虚拟观测（21 态）；配置2：FVM 加速度+角速度 与 IMU 之差（12 态，在线标定）
  - 关键结论：**位置不可观测**（FVM 力不依赖位置，仅大气密度-高度、导航系-位置的弱耦合）；需 Dutch roll 机动激发可观性
  - 配置1 对模型参数误差（±5%/±20%）鲁棒，配置2 大误差下退化
  - 局限：只有仿真；推力模型粗糙（未利用电机/推进系统可测信息）
  - 本仓库拷贝：`Vehicle_Model_Aided_Inertial_Navigation_for_a_UAV_.pdf`

## 1. 五条方法谱系

| 谱系 | 代表工作 | 核心机制 | 状态 |
|---|---|---|---|
| A. 模型/约束辅助（经典滤波） | Koifman & Bar-Itzhack 1999（飞机动力学）；Dissanayake 2001 T-RO（陆车非完整约束）；Bryson 2004（FVM）；Gryte 2021 固定翼（磁力计+空速管+风标） | 硬物理模型 + EKF，虚拟观测 | 2004 后无现代标杆；参考论文仅 38 引用 |
| B. ZUPT | Foxlin 1998 奠基；15 年综述 2020 (arXiv 2008.09208)；C-ZUPT 2025 悬停 (2507.09344)；GNIO 2026 软ZUPT (2603.15281) | 零速/准静态检测 → 速度伪测量 | 最成熟；无人机仅悬停期可用 |
| C. 学习型惯导 | IONet (AAAI'18) → RoNIN (1905.12853) → TLIO (2007.01867) → IDOL (2102.04024) → AirIO 2025 (2501.15659) → AI-IO 2026 (2603.00597) | 深度网络回归位移/速度/姿态 | **当前性能 SOTA 主力** |
| D. 因子图/图优化 | IMU 预积分 (Forster)；Constrained FGO 2025 行人 (2505.08229)；ZUPT-FGO 轮式 (2112.07176) | 全局平滑，榨取全部历史约束 | 行人/水下有；**UAV 上空白** |
| E. 特殊信息源 | CIO 2019 碰撞→速度伪测量 (1909.00079)；IMU 阵列 2026 (2606.29271)；磁力计/气压计/空速管 | 非位置观测的信息通道 | 分散，未联合 |

## 2. SOTA 性能数字

**当前纯 IMU 惯导 SOTA = AI-IO（2026-02，arXiv 2603.00597）**：
- 输入：IMU + **转子转速** + 姿态；CNN 去噪 + Transformer + 不确定性 EKF
- 数据集：Blackbird / VID / DIDO（含 rotor speed 的公开集）+ 自建高机动四旋翼数据集
- ATE：0.86 m vs AirIO 1.76 m（自采高速集）；比 SOTA 提升 44-51%（ATE）
- 作者自认："现有学习型仍缺乏对所学科物理模型的原理性表述"

**基线条目（EuRoC ATE，论文正文表格）**：RoNIN ≈ 6-8 m；TLIO ≈ 5-7 m；AirIO-EKF ≈ 2-4 m。

**AirIO（2025-01）核心贡献**：**保留体坐标系 + 重力分量**（不转全局系）→ ATE 提升 73%；显式编码姿态 → +23.8%；AirIMU 修正 + 不确定性 EKF。

**GNIO（2026-03）**：Gated Prediction Head = 可微软 ZUPT（位移分解为大小+方向门控），4 基准漂移 −60.21%。

**C-ZUPT（2025-07，海法大学）**：不确定性阈值识别准静态平衡 → 空中悬停 ZUPT；多旋翼 LQG 控制协同；场景明确含 delivery；悬停时阻力≈0（二次项）故零速假设成立。

## 3. 三个关键判断

1. 性能 SOTA 全在学习型；但 AI-IO 的 rotor speed 是"学"进去的，不是物理"推"出来的
2. 经典约束路线的现代标杆缺失：C-ZUPT 只覆盖悬停；固定翼停在 2021 仿真；**无人机上"多约束+物理模型+因子图"无人系统化做过**
3. 趋势 = 约束 × 学习融合，但每篇只取一个约束，没有"约束库 + 调度器 + 信息量度量"的框架

## 4. 空白点（按竞争力排序）

| # | 机会 | 与 AI-IO 差异化 |
|---|---|---|
| ① | 推进系统动力学作为**原理性硬约束**（电机转速→推力/力矩→加速度，写进滤波/因子图而非学进网络） | 物理可解释、可证明、不依赖训练分布 |
| ② | 多约束库 + **飞行模式感知调度**（悬停ZUPT/巡航模型约束/起降） | 随模式自动激活，全长飞行漂移抑制 |
| ③ | **因子图 + 无人机多约束**（全局平滑榨取历史） | Constrained FGO 做过行人，UAV 无人做 |
| ④ | **约束信息量定式化**（Fisher 信息/CRLB/冲突软权重） | "榨干信息"的直接题面，理论贡献 |

## 5. 关键论文清单

| 论文 | arXiv | 备注 |
|---|---|---|
| AI-IO: Aerodynamics-Inspired Real-Time IO for Quadrotors | 2603.00597 | 当前 SOTA，竞争核心（全文见 `papers/`） |
| AirIO: Learning IO with Enhanced IMU Feature Observability | 2501.15659 | UAV 学习惯导前 SOTA |
| C-ZUPT: Stationarity-Aided Aerial Hovering | 2507.09344 | 悬停 ZUPT |
| GNIO: Gated Neural Inertial Odometry | 2603.15281 | 可微软 ZUPT |
| EqNIO: Subequivariant Neural IO | 2408.06321 | 等变设计 |
| RIOT: Recursive IO Transformer | 2303.01641 | Transformer 惯导 |
| IONext | 2507.17089 | CNN+Transformer 混合 |
| Constrained FGO for Networked Pedestrian INS | 2505.08229 | 约束因子图（行人） |
| RoNIN | 1905.12853 | 学习惯导基准 |
| TLIO | 2007.01867 | 位移+不确定性紧耦合 |
| IDOL | 2102.04024 | 两阶段姿态+位置 |
| Learned IO for Autonomous Drone Racing | 2210.15287 | 无人机学习惯导+推力 |
| Minimization of GNSS-Denied INS Errors for Fixed Wing UAV | 2108.05188 | 固定翼无GNSS（磁力计+空速管+风标） |
| ZUPT Aided GNSS FGO (Wheeled Robots) | 2112.07176 | ZUPT+因子图 |
| Fifteen Years of Progress at Zero Velocity | 2008.09208 | ZUPT 综述 |
| Contact IO: Collisions are your Friends | 1909.00079 | 碰撞→速度伪测量 |
| Robust EKF with Massive MEMS IMU Array (RISAF) | 2606.29271 | IMU 阵列 |
| OxIOD | 1809.07491 | 学习惯导数据集 |

## 6. 数据资产

- AI-IO 自建"高机动四旋翼数据集"（IMU+rotor speed+真值位姿）是其贡献之一——说明该方向**数据是关键门槛**
- antwork 真实物流数据（电机转速/任务剖面/起降悬停模式）是同级甚至更优资源，且可支撑工业场景叙事

## 7. AI-IO 深读笔记（2026-08-26 精读）

**档案**：上海交大（导航与位置服务重点实验室，Danping Zou 团队）；arXiv 2603.00597（2026-02-28）；代码+数据开源 github.com/SJTU-ViSYS-team/AI-IO。

**方法链**：
1. 叶素-动量理论(BEM) + IMU 测量模型推导观测方程：ã = **ω_m²·v_x（诱导阻力）** − k1·v（线性气动阻）− k2·v²（二次气动阻）+ 推力项 + 偏置 + 噪声
2. 结论：**转子转速是速度可观测性的必要条件**（无转速则观测不完备）→ 这也是它相对 AirIO（纯 IMU）提升的理论依据
3. 网络：CNN（去噪/局部特征，acc/gyro/rpm 三通道）+ **Transformer**（时序依赖/气动建模）+ 双 FC 输出速度+不确定性；Huber + NLL 损失
4. EKF：误差状态 SO(3)，观测 = 网络预测的体坐标系速度（回归到 EKF 20 Hz）

**实验**：自建平台（BMI270+ESC双向Dshot+Vicon，3档速度5/8/14m/s手动+10序列自主，共13km/2636s）；DIDO 对比；实时部署（Radxa Zero3W，8.9ms推理+20Hz EKF，闭环）。

**性能**：DIDO：AI-IO-EKF ATE 3.876 vs AirIO-EKF 5.972（−35.1%）；自采：AVE 0.237 vs 0.556（−57.4%）、ATE 1.809 vs 2.559（−29.3%）；IMU预积分 ATE 102m（崩塌）、IMO 61m（非固定轨迹失效）；vs T265 VIO：正常光照相当，灭灯后 VIO 发散而 AI-IO 保持。消融：rotor speed −36.9%、Transformer −22.4%、CNN 去噪 −23.9%。7 分钟微调即可迁移新平台。

**局限（= 用户机会）**：
| 局限 | 说明 | 用户切入点 |
|---|---|---|
| **无风假设** | 气动力=相对气流速度，无风时才等价机体速度；室外/物流场景不成立 | 风场估计+约束联合 |
| **位置仍靠积分** | EKF 观测只是体速，位置无直接观测，长时漂移未消除 | 约束调度/因子图 |
| **序列短** | 最长约 300 s；物流飞行 30-60 min 未验证 | 长航时漂移 |
| **无模式约束** | 未用悬停 ZUPT/起降等飞行模式信息 | 模式感知约束库 |
| **学习式黑箱** | 物理系数 k_i 是隐含学的，不是显式估计；需真值训练 | 硬物理模型/PINN |
| **室内为主** | 验证在室内低光环境，室外风/气流未测 | 工业真实数据 |
| **未用其他机上传感器** | 磁力计/气压计/空速管未用 | 多渠道信息 |

**作者自认未来方向**：与物理先验更紧密的网络（PINN）、与 VIO 等多传感器框架集成。

## 8. AirIO 深读笔记（2026-08-26 精读）

**档案**：CMU Robotics Institute（Qiu/Xu/Chen/Zhao/Scherer）+ Penn State（Junyi Geng）；**IEEE RA-L 2025**（2025-02 投稿、2025-06-01 接收）；项目页 air-io.github.io。

**核心洞察（"表示即信息"）**：
- 现有做法把 IMU 转到全局系/HACF/去重力 → 姿态与运动力**非线性纠缠**、重力成为"不贡献信息的常数"
- body-frame + 保留重力（式1）：B_a = B_Fnet/m + Rᵀ·G_g，姿态耦合在重力中、与运动力**线性组合** → 网络易学
- 证据：PCA（Blackbird top-5 主成分覆盖 95% vs global 需 8；Pegasus 15 vs 40）+ t-SNE（body 系速度聚类可分、global 纠缠）
- **去掉重力项 RᵀGg → ATE +90%（EuRoC）/+107%（Pegasus）/+360.5%（Blackbird Unseen）**——重力项就是姿态信息载体

**方法**：body IMU + 显式姿态编码（so(3) Lie 代数 + CNN）→ 双向 GRU → 双 MLP（体速+协方差）；Huber+NLL；训练用真值姿态、推理用 EKF 姿态；AirIMU（CMU 2023，学习型 IMU 修正+不确定性）做预处理并入 EKF 过程噪声；误差状态 EKF（SO(3)），观测=网络预测体速；窗口 5 s（1000 帧 @200 Hz）。

**性能（基准数字）**：
- Blackbird SEEN：AirIO-EKF **ATE 0.403 m** vs TLIO 1.189、IMO 0.929、RoNIN 2.193、纯预积分 27.5
- Blackbird **UNSEEN**（泛化）：AirIO **1.309 m** vs IMO 9.015（**严重过拟合**）、RoNIN 13.49、TLIO 3.478
- EuRoC：AirIO-EKF **3.177 m** vs TLIO 6.969、RoNIN 6.75
- vs VIO（附录表 VIII）：EuRoC 上 OpenVINS 优势 64.1%（1.24 vs 3.18）；**Blackbird 上 AirIO 反超 OpenVINS（0.403 vs 0.97）**——8 m/s 峰值运动模糊打垮视觉
- 消融：body vs global 平均 +66.7%；+姿态编码再 −23.8%；模型可压至 0.175 MB（body 退化 44% vs global 81.8%）
- 实时：RTX 2060 28.9 ms；Jetson AGX Orin 74.2 ms

**局限**：
1. 只用 IMU（卖点也是天花板）——无转子转速/推力，气动靠网络硬学 → AI-IO 正是抓住这点补上 rotor speed
2. 数据集全室内/仿真（Blackbird 动捕室、EuRoC 室内、Pegasus 理想仿真）——**无室外风扰**
3. 序列 2-4 分钟/条，无长航时
4. 连续高速轨迹，**无悬停/静止期 → ZUPT 类约束无用武之地也没利用**
5. 训练需要真值姿态（Vicon）；学习型黑箱
6. 实时推理 74 ms 级（Orin），成本偏高

**与 AI-IO 的争论（= 用户的机会）**：AirIO 主张"纯 IMU 即可、不需要额外传感器"；AI-IO 反驳"无转子转速则速度观测不完备"。**这个争论的裁决者正是物理可观测性理论**——用户可以做的工作：给出"何时需要转子转速/哪些约束把观测性补到哪一级"的定量分析 + 用物理硬约束取代网络隐含学习。
