# Buck 变换器电压闭环控制补偿器设计

本项目使用 MATLAB/Simulink 纯基础库，搭建了 Buck 变换器 CCM 平均模型，
并设计 Type II 补偿器实现稳定的电压闭环控制。最终实现：

- 输出电压稳定在 5V（误差 <0.5%）
- 调节时间约 97ms
- 无超调、无振荡

---

## 系统参数

| 参数 | 值 |
|------|-----|
| 输入电压 Vin | 12 V |
| 输出电压 Vref | 5 V |
| 电感 L | 100 μH |
| 电容 C | 1000 μF |
| 负载 R | 5 Ω |
| 开关频率 | 100 kHz（等效平均模型） |

理论稳态占空比：
\[
D = \frac{V_{out}}{V_{in}} = \frac{5}{12} \approx 0.4167
\]

---

## Simulink 模型结构

```
Vref ──→ Σ ──→ 补偿器(零点-极点网络) ──→ 积分器 ──→ 限幅 ──→ 乘法器 ──→ LC滤波器 ──→ Vout
              ↑                                            ↑
              └────────────── 反馈电压 ←────────────────────┘
```

模型完全使用 Simulink 基础库模块，无需附加工具箱：

| 模块 | 库路径 | 参数 |
|------|--------|------|
| Constant (Vref) | Simulink/Sources | 5 |
| Constant (Vin) | Simulink/Sources | 12 |
| Sum | Simulink/Math Operations | `+ -` |
| Transfer Fcn (补偿器) | Simulink/Continuous | `[100, 314200]` / `[1, 62832]` |
| Integrator | Simulink/Continuous | Gain=1, IC=0 |
| Saturation | Simulink/Discontinuities | [0, 0.9] |
| Product | Simulink/Math Operations | 2 |
| Transfer Fcn (LC) | Simulink/Continuous | `[1]` / `[1e-7, 2e-5, 1]` |

---

## Type II 补偿器设计

补偿器传递函数：

\[
G_c(s) = K \cdot \frac{1 + s/\omega_z}{s(1 + s/\omega_p)}
\]

零点放在 LC 谐振频率附近：

\[
f_{LC} = \frac{1}{2\pi\sqrt{LC}} \approx 503 \text{ Hz}
\]

选择：

- 零点 \( f_z = 500 \text{ Hz} \)
- 高频极点 \( f_p = 10 \text{ kHz} \)
- 增益 \( K = 100 \)

> ⚠️ **重要经验**：
> Simulink 的 Transfer Fcn 模块分母常数项不能为 0。
> 因此积分器必须单独用 Integrator 模块，不能与 Transfer Fcn 写在一起。

---

## 调试过程与踩坑记录

### 1️⃣ 开环正常，闭环输出飞到 70V

- **原因**：补偿器分母包含纯积分项，Transfer Fcn 无法正确表示 \(1/s\)
- **解决**：将积分环节拆分为独立 Integrator 模块

### 2️⃣ 闭环后发散振荡

- **原因**：增益 \(K=500\) 过大，穿越频率冲过 LC 谐振点，导致负相位裕度
- **解决**：降低 K，从小到大逐步尝试，最终 K=100

### 3️⃣ 响应极慢，100ms 只爬到 2V

- **原因**：Integrator 的 Gain 被误改成 0.03
- **解决**：检查 Integral Gain 必须为 1

### 4️⃣ 反馈极性确认技巧

- 断开反馈，给固定占空比 0.42，验证输出为 5V
- 将参考电压设为 0，如果输出收敛到 0，说明负反馈正确

---

## 仿真结果

运行 `Buck_Converter_Control.slx`，仿真 0.5s 后：

| 指标 | 实测值 |
|------|--------|
| 稳态输出电压 | 4.984 V |
| 稳态占空比 | 0.418 |
| 调节时间 | 97 ms |
| 超调量 | 0% |
| 振荡 | 无 |

波形截图见 `figures/` 目录。

---

## 文件说明

```
Buck-Converter-Control-Design/
├── README.md                     # 本文件
├── Buck_Converter_Control.slx    # Simulink 模型
├── figures/                      # 仿真波形截图
│   ├── startup_response.png
│   └── steady_state.png
└── docs/
    └── design_notes.md           # 详细设计笔记（可选）
```

---

## 如何使用

1. 克隆本仓库
2. 用 MATLAB R2021b 或更新版本打开 `.slx` 模型
3. 点击运行（仿真停止时间 0.5s）
4. 打开 Scope 查看输出波形

---

## 许可证

本项目采用 [MIT License](LICENSE)

---

## 参考资料

- R. W. Erickson, *Fundamentals of Power Electronics*
- MathWorks 文档：Simulink Control Design
