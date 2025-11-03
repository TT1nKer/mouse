# 电路实现与测试指南
## Circuit Implementation and Testing Guide for Wireless Mouse Charging

---

## 一、系统概述 (System Overview)

### 1.1 目标规格 (Target Specifications)

**应用场景** (Application Scenario)：鼠标垫集成无线充电 (Mouse Pad Integrated Wireless Charging)
- **传输距离** (Transmission Distance)：0-2 cm（近距离，Near Field）
- **输出功率** (Output Power)：100 mW（维持鼠标使用，Maintain Mouse Operation）
- **输入功率** (Input Power)：≤ 1 W（USB供电，USB Powered）
- **系统效率** (System Efficiency)：> 70%（近距离目标，Near-Field Target）
- **工作频率** (Operating Frequency)：13.56 MHz 或 85 kHz
- **发射线圈** (Transmitter Coil)：直径 20 cm（鼠标垫集成，Mouse Pad Integrated）
- **接收线圈** (Receiver Coil)：直径 5-6 cm（鼠标内部，Inside Mouse）

### 1.2 系统架构

```
发射端（鼠标垫，Transmitter - Mouse Pad）：
AC/DC适配器 (AC/DC Adapter) → DC-DC转换 (DC-DC Converter) → 高频逆变器 (High-Frequency Inverter) → 补偿网络 (Compensation Network) → 发射线圈 (Transmitter Coil)

接收端（鼠标，Receiver - Mouse）：
接收线圈 (Receiver Coil) → 补偿网络 (Compensation Network) → 整流电路 (Rectifier Circuit) → DC-DC稳压 (DC-DC Regulator) → 电池管理 (Battery Management) → 负载 (Load)
```

---

## 二、发射端电路实现 (Transmitter Circuit Implementation)

### 2.1 发射端完整电路图

```
┌─────────────────────────────────────────────┐
│          发射端（鼠标垫）                     │
├─────────────────────────────────────────────┤
│                                             │
│  [USB 5V] ──┬── [DC-DC] ── [12V]           │
│             │                                │
│             └── [Buck] ─── [5V MCU]         │
│                                             │
│  [MCU PWM] ──→ [MOSFET驱动] ──→ [全桥逆变]  │
│                                             │
│  [全桥输出] ──→ [LCC补偿] ──→ [Tx线圈]      │
│                                             │
│  [电流检测] ──→ [MCU ADC] (保护)            │
│  [温度检测] ──→ [MCU ADC] (保护)            │
│                                             │
└─────────────────────────────────────────────┘
```

### 2.2 核心组件设计

#### 2.2.1 高频逆变器（全桥）

**电路拓扑**：
```
[DC+] ── [Q1] ──┬── [Tx线圈+补偿] ──┬── [Q2] ── [DC-]
        |                          |
       [Q3] ──┬────────────────┬── [Q4]
        |      |                |
      [DC-]  [DC+]            [DC+]
```

**器件选择** (Component Selection)：
- **MOSFET**: IRF540N 或 SiC MOSFET（高效，High Efficiency）
  - V_DS (Drain-Source Voltage): > 50V
  - I_D (Drain Current): > 10A
  - R_DS(on) (On-Resistance): < 0.1Ω
- **驱动芯片** (Driver IC): IR2110 或 TC4427
  - 双路驱动 (Dual-Channel Driver)，死区控制 (Dead Time Control)

**设计公式**：

**开关频率** (Switching Frequency)：
```
f_switch = f_resonant = f₀ = 1 / (2π × √(L × C))
```

**死区时间** (Dead Time)：
```
t_deadtime = 100-500 ns (防止直通，Prevent Shoot-Through)
```

**开关损耗估算** (Switching Loss Estimation)：
```
P_switch = 2 × (C_oss × V² × f_switch)
其中 (Where)：
- C_oss: MOSFET输出电容 (MOSFET Output Capacitance)
- V: 开关电压 (Switching Voltage)
- f_switch: 开关频率 (Switching Frequency)
```

**导通损耗** (Conduction Loss)：
```
P_conduction = I_rms² × R_DS(on) × 2
其中 (Where)：
- I_rms: 线圈RMS电流 (Coil RMS Current)
- R_DS(on): MOSFET导通电阻 (MOSFET On-Resistance)
```

#### 2.2.2 LCC补偿网络

**电路结构**：
```
[逆变器] ── [Lf] ── [Cf] ── [L₁ || C₁] ── [发射线圈]
```

**设计公式**：

**1. 补偿电感 Lf**
```
Lf = k × L₁
其中：
- k = 0.1-0.3 (经验系数，可优化)
- L₁ = 发射线圈电感
```

**2. 补偿电容 Cf**
```
Cf = 1 / (4π² × f₀² × Lf)
```

**3. 谐振电容 C₁**
```
C₁ = (1 - k) / (4π² × f₀² × L₁)
或
C₁ = 1 / (4π² × f₀² × L₁) - Cf × Lf / L₁
```

**4. LCC补偿特点**
- 恒流输出特性
- 对负载变化鲁棒
- 适合距离变化场景

**设计示例**（13.56 MHz，L₁ = 5 µH）：
```
假设 k = 0.2:
Lf = 0.2 × 5 µH = 1 µH
Cf = 1 / (4π² × (13.56×10⁶)² × 1×10⁻⁶) = 137.5 pF
C₁ = (1-0.2) / (4π² × (13.56×10⁶)² × 5×10⁻⁶) = 22 pF
```

#### 2.2.3 发射线圈设计

**线圈参数** (Coil Parameters)：
- **类型** (Type): PCB线圈 (PCB Coil)（适合薄型设计，Suitable for Thin Design）
- **直径** (Diameter): D = 20 cm
- **层数** (Layers): 1-2层（多层提高电感，Multiple Layers Increase Inductance）
- **线宽** (Trace Width): 0.5-1 mm
- **间距** (Spacing): 0.5 mm

**设计公式**：

**1. 电感计算**（平面螺旋线圈）
```
L ≈ (μ₀ × N² × r_avg × ln(8r_avg/a)) / 2

其中：
- μ₀ = 4π×10⁻⁷ H/m（真空磁导率）
- N = 匝数
- r_avg = 线圈平均半径 = (r_outer + r_inner) / 2
- a = 导线半径（PCB中为线宽/2）
```

**2. 串联电阻（高频损耗）**
```
R_AC = R_DC × (1 + √(d/δ))
其中：
- R_DC = 直流电阻
- d = 导体厚度
- δ = 趋肤深度 = √(2ρ/(ωμ))
```

**3. 品质因数Q**
```
Q = ωL / R_AC = 2πfL / R_AC
目标：Q > 100
```

**4. 谐振电容**
```
C₁ = 1 / (4π² × f₀² × L₁)
```

**设计示例**（13.56 MHz，目标L₁ = 5 µH）：
```
假设：r_avg = 0.1 m, a = 0.0005 m (1mm线宽)

5×10⁻⁶ ≈ (4π×10⁻⁷ × N² × 0.1 × ln(8×0.1/0.0005)) / 2
求解：N ≈ 8 匝

验证Q值（假设R_AC = 0.1Ω）：
Q = 2π × 13.56×10⁶ × 5×10⁻⁶ / 0.1 ≈ 4260
（实际会更低，因为高频损耗）
```

#### 2.2.4 控制与保护电路

**1. 电流检测** (Current Sensing)
```
使用分流电阻 + 差分放大器 (Using Shunt Resistor + Differential Amplifier)：
V_sense = I_coil × R_shunt × Gain

保护阈值 (Protection Threshold)：
I_max = 1.5 × I_nominal
```

**2. 温度检测** (Temperature Sensing)
```
使用NTC热敏电阻 (Using NTC Thermistor)：
R_T = R_25 × exp(B × (1/T - 1/298.15))

保护阈值 (Protection Threshold)：
T_max = 60-70°C
```

**3. 频率控制** (Frequency Control)
```
使用MCU PWM或专用频率发生器 (Using MCU PWM or Dedicated Frequency Generator)：
- 精确控制开关频率 (Precise Switching Frequency Control)
- 可调频率（频率跟踪优化，Adjustable Frequency - Frequency Tracking Optimization）
```

---

## 三、接收端电路实现 (Receiver Circuit Implementation)

### 3.1 接收端完整电路图

```
┌─────────────────────────────────────────────┐
│          接收端（鼠标内部）                    │
├─────────────────────────────────────────────┤
│                                             │
│  [Rx线圈] ──→ [S-S补偿] ──→ [全桥整流]      │
│                                             │
│  [整流输出] ──→ [滤波] ──→ [DC-DC] ──→ [3.7V]│
│                                             │
│  [电池管理] ──→ [电池] ──→ [负载]            │
│                                             │
│  [电压检测] ──→ [MCU ADC]                   │
│  [电流检测] ──→ [MCU ADC]                   │
│                                             │
└─────────────────────────────────────────────┘
```

### 3.2 核心组件设计

#### 3.2.1 S-S补偿网络

**电路结构**：
```
[接收线圈] ── [C₂] ── [整流电路]
```

**设计公式**：

**1. 谐振电容 C₂**
```
C₂ = 1 / (4π² × f₀² × L₂)

其中：
- L₂ = 接收线圈电感
- f₀ = 谐振频率（与发射端相同）
```

**2. 补偿特点**
- 简单结构
- 电流大，适合功率传输
- 对距离敏感（但近距离影响小）

**设计示例**（13.56 MHz，L₂ = 1 µH）：
```
C₂ = 1 / (4π² × (13.56×10⁶)² × 1×10⁻⁶) = 137.5 pF
```

#### 3.2.2 接收线圈设计

**线圈参数**：
- **类型**: PCB线圈（薄型）
- **直径**: D = 5-6 cm
- **层数**: 2-4层（多层提高电感）
- **线宽**: 0.3-0.5 mm
- **间距**: 0.3 mm

**设计公式**（同发射端）：
```
L₂ ≈ (μ₀ × N² × r_avg × ln(8r_avg/a)) / 2

目标：
- L₂ = 1-2 µH（13.56 MHz）
- Q₂ > 100
```

#### 3.2.3 整流电路

**全桥整流**：
```
[L₂+] ── [D1] ──┬── [+] ──→ [滤波]
     |           |
     └── [D3] ──┴── [-] ──→ [GND]
     |
[L₂-] ── [D2] ──┬── [+] ──→ [滤波]
     |           |
     └── [D4] ──┴── [-] ──→ [GND]
```

**器件选择** (Component Selection)：
- **二极管** (Diode): 肖特基二极管 (Schottky Diode)（低压降，Low Forward Drop）
  - 示例 (Examples): BAT54S, MBR0540
  - V_f (Forward Voltage): < 0.3V @ I_forward
  - V_RRM (Peak Reverse Voltage): > 20V

**设计公式**：

**1. 整流效率**
```
η_rectifier = (V_out - 2×V_f) / V_out
其中：
- V_out: 整流后电压
- V_f: 二极管正向压降
```

**2. 二极管损耗**
```
P_diode = I_avg × V_f × 2（全桥）
```

#### 3.2.4 DC-DC稳压电路

**Buck转换器**（降压）：
```
[整流输出] ── [Buck IC] ── [3.7V稳定输出]
```

**器件选择** (Component Selection)：
- **Buck IC** (Buck Converter IC): TPS63000, LM2596, MP2307
- **效率** (Efficiency): > 85%
- **输入范围** (Input Range): 5-12V
- **输出** (Output): 3.7V（锂电池，Lithium Battery）

**设计公式**：

**1. 占空比**
```
D = V_out / V_in
```

**2. 电感选择**
```
L_buck = (V_in - V_out) × D / (ΔI_L × f_switch)
其中：
- ΔI_L = 纹波电流（通常取I_out的20-30%）
- f_switch = Buck开关频率
```

**3. 输出电容**
```
C_out = ΔI_L / (8 × f_switch × ΔV_out)
其中：
- ΔV_out = 允许的电压纹波
```

#### 3.2.5 电池管理

**充电管理IC** (Charging Management IC)：
- TP4056, MCP73831（单节锂电池，Single-Cell Lithium Battery）
- 恒流恒压充电 (Constant Current/Constant Voltage Charging, CC/CV Charging)
- 过充/过放保护 (Overcharge/Over-Discharge Protection)

**设计公式**：

**1. 充电电流**
```
I_charge = (V_rectifier - V_battery) / R_charge
限制：I_charge < C/2（C = 电池容量）
```

**2. 电池容量需求**
```
假设：
- 鼠标功耗：100 mW（使用中）
- 电池电压：3.7V
- 工作时间：8小时/天

电池容量 = (100 mW / 3.7V) × 8h / 0.8（效率）
         ≈ 270 mAh

选择：500 mAh（安全余量）
```

---

## 四、关键设计公式汇总 (Key Design Formulas)

### 4.1 谐振频率 (Resonant Frequency)

```
f₀ = 1 / (2π × √(L × C))
```

### 4.2 耦合系数 (Coupling Coefficient)

**互感计算** (Mutual Inductance Calculation)（两个平行圆形线圈，同轴，Two Parallel Circular Coils, Coaxial）：
```
M = (μ₀ × π × r₁² × r₂² × N₁ × N₂) / (2 × (r² + d²)^(3/2))

当 d = 0（完全同轴，Perfectly Coaxial）：
M ≈ (μ₀ × π × r₁² × r₂² × N₁ × N₂) / (2 × r³)

其中 (Where)：
- r₁, r₂ = 线圈半径 (Coil Radius)
- r = 轴向距离 (Axial Distance)
- d = 径向偏移 (Radial Offset)
- N₁, N₂ = 匝数 (Number of Turns)
```

**耦合系数** (Coupling Coefficient)：
```
k = M / √(L₁ × L₂)
```

### 4.3 品质因数 (Quality Factor)

```
Q = ωL / R = 2πfL / R
```

### 4.4 传输效率 (Transfer Efficiency)

**谐振耦合效率（S-S补偿，Resonant Coupling Efficiency - S-S Compensation）**：
```
η = k² × Q₁ × Q₂ / ((1 + k² × Q₁ × Q₂) × (1 + R_load/(ωL₂)))

简化（假设负载匹配，Simplified - Assuming Load Matching）：
η ≈ k² × Q₁ × Q₂ / (1 + k² × Q₁ × Q₂)

其中 (Where)：
- k = 耦合系数 (Coupling Coefficient)
- Q₁, Q₂ = 发射/接收线圈品质因数 (Transmitter/Receiver Coil Quality Factor)
- R_load = 负载电阻 (Load Resistance)
```

### 4.5 功率传输 (Power Transfer)

**最大功率传输**（阻抗匹配，Maximum Power Transfer - Impedance Matching）：
```
R_load_opt = ωL₂ / Q₂

P_max = V_coil² / (4 × R_load_opt)
```

### 4.6 LCC补偿参数 (LCC Compensation Parameters)

**补偿电感** (Compensation Inductor)：
```
Lf = k × L₁  (k = 0.1-0.3)
```

**补偿电容** (Compensation Capacitor)：
```
Cf = 1 / (4π² × f₀² × Lf)
```

**谐振电容** (Resonant Capacitor)：
```
C₁ = (1 - k) / (4π² × f₀² × L₁)
```

### 4.7 线圈设计 (Coil Design)

**平面螺旋线圈电感** (Planar Spiral Coil Inductance)：
```
L ≈ (μ₀ × N² × r_avg × ln(8r_avg/a)) / 2

其中 (Where)：
- r_avg = (r_outer + r_inner) / 2
- a = 导线半径 (Wire Radius)
```

**串联电阻（高频）** (Series Resistance - High Frequency)：
```
R_AC = R_DC × (1 + √(d/δ))

其中 (Where)：
- δ = 趋肤深度 (Skin Depth) = √(2ρ/(ωμ))
- ρ = 电阻率 (Resistivity)
- d = 导体厚度 (Conductor Thickness)
```

---

## 五、测试程序与验证 (Testing Procedures)

### 5.1 预设计验证测试

#### 5.1.1 线圈参数测试

**测试设备** (Test Equipment)：
- LCR表 (LCR Meter)（测量电感和Q值，Measure Inductance and Q Factor）
- 网络分析仪 (Network Analyzer)（可选，精确测量，Optional, Precise Measurement）

**测试步骤**：

**1. 电感测量**
```
步骤：
1. 使用LCR表在谐振频率f₀附近测量L
2. 比较测量值与计算值
3. 误差应在±10%以内

目标：
- L₁ = 5 µH ± 10%（发射）
- L₂ = 1-2 µH ± 10%（接收）
```

**2. Q值测量**
```
步骤：
1. 使用LCR表测量Q值
2. 或使用网络分析仪测量S11，计算Q
3. Q = f₀ / Δf_3dB

目标：
- Q₁, Q₂ > 100
```

**3. 谐振频率验证**
```
步骤：
1. 连接谐振电容C₁/C₂
2. 使用网络分析仪扫频
3. 找到阻抗最小的频率（谐振频率）
4. 应与设计值f₀一致

目标：
- f_measured = f_design ± 1%
```

**通过标准**：
- ✅ 电感误差 < ±10%
- ✅ Q值 > 100
- ✅ 谐振频率误差 < ±1%

#### 5.1.2 补偿网络测试

**测试设备** (Test Equipment)：
- 网络分析仪 (Network Analyzer)
- 信号发生器 (Signal Generator)
- 示波器 (Oscilloscope)

**测试步骤**：

**1. 补偿网络频率响应**
```
步骤：
1. 连接补偿网络（LCC或S-S）
2. 使用网络分析仪测量S21（传输系数）
3. 验证在f₀处有峰值响应
4. 测量带宽和Q值

目标：
- 峰值频率 = f₀ ± 1%
- 3dB带宽 < 1% f₀
```

**2. 阻抗匹配验证**
```
步骤：
1. 测量输入阻抗Z_in（发射端）
2. 测量输出阻抗Z_out（接收端）
3. 计算匹配度

目标：
- Z_in ≈ 50Ω（与源阻抗匹配）
- Z_out ≈ R_load（与负载匹配）
```

**通过标准**：
- ✅ 谐振频率准确
- ✅ 带宽满足要求
- ✅ 阻抗匹配良好

### 5.2 发射端独立测试

#### 5.2.1 逆变器测试

**测试设备** (Test Equipment)：
- 示波器 (Oscilloscope)（至少100 MHz带宽，At Least 100 MHz Bandwidth）
- 电流探头 (Current Probe)
- 功率计 (Power Meter)

**测试步骤**：

**1. 开关频率验证**
```
步骤：
1. 连接示波器到MOSFET栅极
2. 测量PWM频率
3. 测量占空比和死区时间

目标：
- f_switch = f₀ ± 1%
- 占空比 = 50% ± 2%
- 死区时间 = 100-500 ns
```

**2. 开关波形测试**
```
测试项：
- 上升时间：t_r < 50 ns
- 下降时间：t_f < 50 ns
- 振铃：< 20% V_peak
- 过冲：< 10% V_peak
```

**3. 开关损耗测量**
```
步骤：
1. 测量V_DS和I_DS
2. 计算 P_switch = ∫(V_DS × I_DS) dt × f_switch
3. 或使用功率分析仪直接测量

目标：
- P_switch < 5% P_total
```

**4. 导通损耗测量**
```
步骤：
1. 测量I_rms
2. 计算 P_cond = I_rms² × R_DS(on) × 2

目标：
- P_cond < 10% P_total
```

**通过标准**：
- ✅ 开关频率准确
- ✅ 波形质量良好（无过冲、振铃）
- ✅ 总损耗 < 15% P_total

#### 5.2.2 线圈电流测试

**测试设备**：
- 电流探头
- 示波器

**测试步骤**：

**1. 线圈电流幅度**
```
步骤：
1. 使用电流探头测量线圈电流I_coil
2. 验证I_coil在安全范围内

目标：
- I_coil_RMS < 2A（根据设计）
- 无异常尖峰
```

**2. 谐振验证**
```
步骤：
1. 测量电流相位相对于电压
2. 在谐振频率下，电流与电压同相
3. 电流幅度最大

目标：
- 相位差 < 10°（接近同相）
- 电流幅度符合设计
```

**通过标准**：
- ✅ 电流在安全范围
- ✅ 谐振状态正确

#### 5.2.3 保护功能测试

**测试步骤**：

**1. 过流保护**
```
步骤：
1. 逐渐增加负载
2. 当I_coil > I_max时，系统应关闭或限流
3. 记录保护阈值

目标：
- I_protect = 1.5 × I_nominal ± 5%
```

**2. 温度保护**
```
步骤：
1. 加热系统（使用热风枪）
2. 当温度 > T_max时，系统应关闭
3. 记录保护温度

目标：
- T_protect = 60-70°C
```

**3. 故障检测**
```
步骤：
1. 断开负载（空载）
2. 系统应检测到故障并保护
3. 短路负载
4. 系统应检测到故障并保护

目标：
- 保护功能正常
- 无器件损坏
```

**通过标准**：
- ✅ 过流保护准确
- ✅ 温度保护准确
- ✅ 故障检测正常

### 5.3 接收端独立测试

#### 5.3.1 整流电路测试

**测试设备** (Test Equipment)：
- 示波器 (Oscilloscope)
- 电压表 (Voltmeter)
- 电流表 (Ammeter)

**测试步骤**：

**1. 整流效率**
```
步骤：
1. 使用信号发生器产生交流信号（频率f₀）
2. 连接整流电路
3. 测量输入功率P_in和输出功率P_out
4. 计算效率 η = P_out / P_in

目标：
- η_rectifier > 85%
```

**2. 输出电压纹波**
```
步骤：
1. 测量整流输出直流电压
2. 测量交流纹波分量
3. 计算纹波百分比

目标：
- 纹波 < 10% V_out（滤波前）
- 纹波 < 1% V_out（滤波后）
```

**3. 二极管压降**
```
步骤：
1. 测量每个二极管的V_f
2. 在不同电流下测量

目标：
- V_f < 0.3V @ I_rated
```

**通过标准**：
- ✅ 整流效率 > 85%
- ✅ 纹波满足要求
- ✅ 二极管压降低

#### 5.3.2 DC-DC转换器测试

**测试步骤**：

**1. 效率测试**
```
步骤：
1. 在不同负载下测量效率
2. 轻载（10%）、中载（50%）、重载（100%）

目标：
- η_DC-DC > 85%（全负载范围）
```

**2. 输出电压精度**
```
步骤：
1. 在不同负载下测量输出电压
2. 计算稳压精度

目标：
- V_out = V_set ± 2%
```

**3. 动态响应**
```
步骤：
1. 负载阶跃变化（10% → 90%）
2. 测量输出电压恢复时间

目标：
- 恢复时间 < 100 µs
- 过冲 < 5% V_out
```

**通过标准**：
- ✅ 效率 > 85%
- ✅ 输出电压稳定
- ✅ 动态响应良好

### 5.4 系统集成测试

#### 5.4.1 功率传输效率测试

**测试设备**：
- 功率计（输入和输出）
- 电压表、电流表
- 电子负载

**测试步骤**：

**1. 距离-效率关系**
```
步骤：
1. 将接收端放在不同距离（0.5cm, 1cm, 2cm）
2. 测量输入功率P_in和输出功率P_out
3. 计算效率 η = P_out / P_in
4. 记录数据

目标：
- 距离0-2cm：η > 70%
- 距离2-5cm：η > 50%
```

**测试表格**：
| 距离 (cm) | P_in (W) | P_out (W) | 效率 (%) | 通过/失败 |
|-----------|----------|-----------|----------|-----------|
| 0.5       |          |           |          |           |
| 1.0       |          |           |          |           |
| 2.0       |          |           |          |           |
| 5.0       |          |           |          |           |

**2. 功率-效率关系**
```
步骤：
1. 固定距离（1cm）
2. 改变输出功率（10mW, 50mW, 100mW, 200mW）
3. 测量效率

目标：
- 在100mW时，η > 70%
```

**3. 错位容忍度测试**
```
步骤：
1. 接收端与发射端中心对齐（效率基准）
2. 径向偏移1cm, 2cm, 3cm
3. 测量效率变化

目标：
- 偏移2cm内：效率下降 < 20%
```

**通过标准**：
- ✅ 目标距离效率 > 70%
- ✅ 错位容忍度满足要求

#### 5.4.2 功率输出能力测试

**测试步骤**：

**1. 最大功率测试**
```
步骤：
1. 逐渐增加负载
2. 测量能稳定输出的最大功率
3. 验证保护功能

目标：
- P_max > 150 mW（留50%余量）
```

**2. 功率稳定性**
```
步骤：
1. 固定负载（100mW）
2. 连续运行1小时
3. 监测输出功率波动

目标：
- 功率波动 < ±5%
- 无异常中断
```

**3. 瞬态响应**
```
步骤：
1. 负载突然变化（0 → 100mW）
2. 测量输出电压恢复
3. 测量恢复时间

目标：
- 恢复时间 < 100 ms
- 无振荡
```

**通过标准**：
- ✅ 最大功率满足要求
- ✅ 功率稳定
- ✅ 瞬态响应良好

#### 5.4.3 温度测试

**测试设备** (Test Equipment)：
- 热成像仪或热电偶 (Thermal Imager or Thermocouple)
- 温度记录仪 (Temperature Logger)

**测试步骤**：

**1. 稳态温度**
```
步骤：
1. 系统满载运行1小时
2. 测量各组件温度
3. 记录最高温度点

目标：
- MOSFET温度 < 60°C
- 线圈温度 < 50°C
- 其他组件 < 70°C
```

**2. 温升测试**
```
步骤：
1. 测量室温下的初始温度
2. 满载运行直到温度稳定
3. 计算温升

目标：
- 温升 < 30°C
```

**通过标准**：
- ✅ 温度在安全范围
- ✅ 无热失效

#### 5.4.4 EMI/EMC测试

**测试设备** (Test Equipment)：
- 频谱分析仪 (Spectrum Analyzer)
- 近场探头 (Near-Field Probe)
- 传导EMI测试设备 (Conducted EMI Test Equipment)

**测试步骤**：

**1. 辐射发射测试**
```
步骤：
1. 使用频谱分析仪测量辐射
2. 在1米距离测量
3. 检查是否超过限制

目标：
- 符合FCC Class B限制
- 不干扰Wi-Fi/蓝牙（2.4 GHz）
```

**2. 传导发射测试**
```
步骤：
1. 测量电源线上的传导发射
2. 检查是否超过限制

目标：
- 符合FCC Class B限制
```

**通过标准**：
- ✅ 辐射发射达标
- ✅ 传导发射达标

#### 5.4.5 长期可靠性测试

**测试步骤**：

**1. 老化测试**
```
步骤：
1. 系统满载运行24小时
2. 监测性能变化
3. 检查是否有性能下降

目标：
- 效率下降 < 2%
- 无故障
```

**2. 循环测试**
```
步骤：
1. 开关循环（开1小时，关30分钟）
2. 重复100次
3. 检查系统稳定性

目标：
- 无故障
- 性能稳定
```

**通过标准**：
- ✅ 长期稳定性良好
- ✅ 无可靠性问题

### 5.5 鼠标集成测试

#### 5.5.1 充电功能测试

**测试步骤**：

**1. 充电速度**
```
步骤：
1. 电池从10%开始充电
2. 测量充电时间到80%
3. 计算充电速度

目标：
- 充电速度 > 50 mAh/小时
- 或：使用中电池维持/增加
```

**2. 充电效率**
```
步骤：
1. 测量输入功率（鼠标垫）
2. 测量实际充电功率（电池）
3. 计算充电效率

目标：
- η_charge > 50%（考虑所有损耗）
```

**3. 电池保护**
```
步骤：
1. 测试过充保护
2. 测试过放保护
3. 测试温度保护

目标：
- 所有保护功能正常
```

**通过标准**：
- ✅ 充电速度满足要求
- ✅ 充电效率 > 50%
- ✅ 电池保护正常

#### 5.5.2 使用场景测试

**测试步骤**：

**1. 正常使用测试**
```
步骤：
1. 鼠标正常使用（移动、点击）
2. 监测电池电量变化
3. 运行8小时

目标：
- 电池电量维持或增加
- 鼠标功能正常
```

**2. 不同表面测试**
```
步骤：
1. 在不同材质表面使用（木桌、玻璃、鼠标垫）
2. 监测功率传输
3. 检查是否有影响

目标：
- 所有表面正常工作
- 效率变化 < 10%
```

**3. 错位测试**
```
步骤：
1. 鼠标在鼠标垫上不同位置
2. 监测充电情况

目标：
- 在充电区域内正常充电
```

**通过标准**：
- ✅ 正常使用场景通过
- ✅ 不同表面兼容
- ✅ 错位容忍度良好

---

## 六、测试报告模板 (Test Report Template)

### 6.1 测试总结

**测试日期**：_____________
**测试人员**：_____________
**系统版本**：_____________

### 6.2 测试结果汇总

| 测试项目 | 目标值 | 实测值 | 状态 | 备注 |
|---------|--------|--------|------|------|
| 线圈电感L₁ | 5 µH ± 10% | | | |
| 线圈电感L₂ | 1-2 µH ± 10% | | | |
| 谐振频率 | f₀ ± 1% | | | |
| Q值 | > 100 | | | |
| 系统效率（1cm） | > 70% | | | |
| 最大功率 | > 150 mW | | | |
| 充电效率 | > 50% | | | |
| 温度 | < 70°C | | | |
| EMI | 符合标准 | | | |

### 6.3 问题记录

**发现问题**：
1. 
2. 
3. 

**解决方案**：
1. 
2. 
3. 

### 6.4 结论

**系统状态**：□ 通过  □ 需要改进  □ 失败

**下一步行动**：
1. 
2. 
3. 

---

## 七、常见问题与调试 (Troubleshooting)

### 7.1 效率太低

**可能原因**：
1. 谐振频率不匹配
2. 耦合系数太低（距离太远或错位）
3. Q值太低
4. 补偿网络不合适

**调试步骤**：
1. 使用网络分析仪测量实际谐振频率
2. 调整电容值，匹配谐振频率
3. 检查线圈位置和对齐
4. 测量并优化Q值
5. 尝试不同的补偿网络（S-S → LCC）

### 7.2 功率不足

**可能原因**：
1. 输入功率不足
2. 线圈电流限制
3. 负载不匹配
4. 损耗太大

**调试步骤**：
1. 测量输入功率，确保足够
2. 检查线圈电流是否达到设计值
3. 优化负载匹配
4. 减少各种损耗（开关损耗、导通损耗等）

### 7.3 发热严重

**可能原因**：
1. 开关损耗大
2. 导通损耗大
3. 线圈损耗大
4. 散热不足

**调试步骤**：
1. 优化死区时间，减少开关损耗
2. 选择低R_DS(on)的MOSFET
3. 使用Litz线降低线圈损耗
4. 改善散热设计

### 7.4 系统不稳定

**可能原因**：
1. 频率漂移
2. 负载变化
3. 温度变化
4. 干扰

**调试步骤**：
1. 使用锁相环（PLL）锁定频率
2. 实现动态频率调谐
3. 优化补偿网络提高鲁棒性
4. 添加滤波和屏蔽

---

## 八、设计检查清单 (Design Checklist)

### 8.1 发射端检查

- [ ] 线圈电感L₁设计正确
- [ ] 谐振电容C₁计算正确
- [ ] LCC补偿参数计算正确
- [ ] MOSFET选择合适（V_DS, I_D, R_DS(on)）
- [ ] 驱动电路设计正确
- [ ] 保护电路（过流、过温）实现
- [ ] 控制电路功能完整
- [ ] PCB布局合理（高频走线短，地平面完整）

### 8.2 接收端检查

- [ ] 线圈电感L₂设计正确
- [ ] 谐振电容C₂计算正确
- [ ] 整流电路效率高
- [ ] DC-DC转换器设计正确
- [ ] 电池管理电路正确
- [ ] 保护功能完整
- [ ] PCB布局合理

### 8.3 系统检查

- [ ] 发射端和接收端谐振频率匹配
- [ ] 耦合系数k满足要求
- [ ] Q值足够高
- [ ] 系统效率满足目标
- [ ] 功率输出满足要求
- [ ] EMI/EMC考虑
- [ ] 温度设计合理
- [ ] 可靠性设计考虑

---

## 九、下一步行动 (Next Steps)

### 9.1 设计阶段完成标准

当以下所有项都通过测试时，可以进入下一阶段：

1. ✅ **预设计验证**：线圈参数、补偿网络都正确
2. ✅ **发射端测试**：逆变器、线圈、保护功能正常
3. ✅ **接收端测试**：整流、DC-DC、电池管理正常
4. ✅ **系统集成测试**：效率、功率、温度都达标
5. ✅ **EMI测试**：符合标准
6. ✅ **可靠性测试**：长期运行稳定

### 9.2 进入下一阶段

完成电路实现和测试后，可以进入：

1. **鼠标集成阶段**：
   - 将接收线圈集成到鼠标PCB
   - 将发射线圈集成到鼠标垫
   - 机械结构设计
   - 整体优化

2. **产品化阶段**：
   - 成本优化
   - 量产准备
   - 认证申请（FCC、CE等）
   - 用户测试

---

## 十、参考资源 (References)

### 10.1 计算公式工具

**在线计算器** (Online Calculators)：
- 线圈电感计算器 (Coil Inductance Calculator)
- 谐振频率计算器 (Resonant Frequency Calculator)
- EMI计算工具 (EMI Calculation Tools)

**MATLAB/Python脚本** (MATLAB/Python Scripts)：
- 可以编写脚本自动计算设计参数 (Can Write Scripts to Automatically Calculate Design Parameters)
- 参数扫描和优化 (Parameter Sweeping and Optimization)

### 10.2 仿真工具

**电路仿真** (Circuit Simulation)：
- LTspice（免费，Free）
- PSIM（电力电子仿真，Power Electronics Simulation）

**电磁场仿真** (Electromagnetic Field Simulation)：
- COMSOL Multiphysics（推荐，Recommended）
- ANSYS HFSS
- OpenEMS（开源，Open Source）

### 10.3 测试设备

**必需设备** (Essential Equipment)：
- LCR表 (LCR Meter)
- 示波器 (Oscilloscope)（100 MHz+）
- 功率计 (Power Meter)
- 网络分析仪 (Network Analyzer)（可选但推荐，Optional but Recommended）

**可选设备** (Optional Equipment)：
- 频谱分析仪 (Spectrum Analyzer)（EMI测试，EMI Testing）
- 热成像仪 (Thermal Imager)（温度测试，Temperature Testing）
- 电子负载 (Electronic Load)（功率测试，Power Testing）

---

## 十一、重要器件与术语表 (Important Components and Terms Glossary)

### 11.1 器件型号 (Component Part Numbers)

#### 11.1.1 功率器件 (Power Devices)

| 中文名称 | English Name | Part Number/Type | 用途 (Application) |
|---------|-------------|-----------------|-------------------|
| MOSFET | MOSFET | IRF540N, IRF540 | 开关管 (Switching Transistor) |
| SiC MOSFET | SiC MOSFET | - | 高效率开关管 (High-Efficiency Switch) |
| 驱动芯片 | Driver IC | IR2110 | MOSFET驱动 (MOSFET Driver) |
| 驱动芯片 | Driver IC | TC4427 | 双路MOSFET驱动 (Dual-Channel MOSFET Driver) |

#### 11.1.2 二极管 (Diodes)

| 中文名称 | English Name | Part Number | 用途 (Application) |
|---------|-------------|------------|-------------------|
| 肖特基二极管 | Schottky Diode | BAT54S | 整流，低电压降 (Rectification, Low Voltage Drop) |
| 肖特基二极管 | Schottky Diode | MBR0540 | 整流，低电压降 (Rectification, Low Voltage Drop) |

#### 11.1.3 DC-DC转换器 (DC-DC Converters)

| 中文名称 | English Name | Part Number | 用途 (Application) |
|---------|-------------|------------|-------------------|
| Buck转换器 | Buck Converter | TPS63000 | 降压转换 (Step-Down Converter) |
| Buck转换器 | Buck Converter | LM2596 | 降压转换 (Step-Down Converter) |
| Buck转换器 | Buck Converter | MP2307 | 降压转换 (Step-Down Converter) |

#### 11.1.4 电池管理 (Battery Management)

| 中文名称 | English Name | Part Number | 用途 (Application) |
|---------|-------------|------------|-------------------|
| 充电管理IC | Charging Management IC | TP4056 | 锂电池充电 (Lithium Battery Charging) |
| 充电管理IC | Charging Management IC | MCP73831 | 锂电池充电 (Lithium Battery Charging) |

#### 11.1.5 传感器 (Sensors)

| 中文名称 | English Name | Part Number/Type | 用途 (Application) |
|---------|-------------|-----------------|-------------------|
| NTC热敏电阻 | NTC Thermistor | - | 温度检测 (Temperature Sensing) |
| 分流电阻 | Shunt Resistor | - | 电流检测 (Current Sensing) |

### 11.2 电路组件术语 (Circuit Component Terms)

| 中文 | English | 说明 (Description) |
|------|---------|-------------------|
| 发射线圈 | Transmitter Coil | 无线功率传输的发射端线圈 (Transmitter-side coil for wireless power transfer) |
| 接收线圈 | Receiver Coil | 无线功率传输的接收端线圈 (Receiver-side coil for wireless power transfer) |
| 补偿网络 | Compensation Network | 补偿电路（LCC、S-S等）(Compensation circuit - LCC, S-S, etc.) |
| 全桥逆变器 | Full-Bridge Inverter | H桥逆变电路 (H-bridge inverter circuit) |
| 整流电路 | Rectifier Circuit | 交流转直流电路 (AC to DC converter circuit) |
| 全桥整流 | Full-Bridge Rectifier | 四二极管整流电路 (Four-diode rectifier circuit) |
| 死区时间 | Dead Time | 防止上下管直通的时间间隔 (Time interval to prevent shoot-through) |
| 谐振频率 | Resonant Frequency | 谐振电路的固有频率 (Natural frequency of resonant circuit) |
| 耦合系数 | Coupling Coefficient | 两个线圈之间的耦合程度 (Degree of coupling between two coils) |
| 品质因数 | Quality Factor (Q) | 衡量谐振电路质量的参数 (Parameter measuring resonant circuit quality) |

### 11.3 测试设备术语 (Test Equipment Terms)

| 中文 | English | 说明 (Description) |
|------|---------|-------------------|
| LCR表 | LCR Meter | 测量电感、电容、电阻的仪器 (Instrument for measuring inductance, capacitance, resistance) |
| 网络分析仪 | Network Analyzer | 测量电路频率响应的仪器 (Instrument for measuring circuit frequency response) |
| 示波器 | Oscilloscope | 显示电信号波形的仪器 (Instrument for displaying electrical signal waveforms) |
| 频谱分析仪 | Spectrum Analyzer | 分析信号频谱的仪器 (Instrument for analyzing signal spectrum) |
| 功率计 | Power Meter | 测量功率的仪器 (Instrument for measuring power) |
| 电流探头 | Current Probe | 测量电流的探头 (Probe for measuring current) |
| 热成像仪 | Thermal Imager | 显示温度分布的仪器 (Instrument for displaying temperature distribution) |
| 热电偶 | Thermocouple | 温度传感器 (Temperature sensor) |
| 电子负载 | Electronic Load | 可编程负载 (Programmable load) |
| 信号发生器 | Signal Generator | 产生测试信号的仪器 (Instrument for generating test signals) |
| 近场探头 | Near-Field Probe | 测量近场EMI的探头 (Probe for measuring near-field EMI) |

### 11.4 技术术语 (Technical Terms)

| 中文 | English | 缩写 (Abbreviation) |
|------|---------|---------------------|
| 开关频率 | Switching Frequency | f_switch |
| 导通损耗 | Conduction Loss | P_cond |
| 开关损耗 | Switching Loss | P_switch |
| 输出电压纹波 | Output Voltage Ripple | ΔV_out |
| 占空比 | Duty Cycle | D |
| 负载电阻 | Load Resistance | R_load |
| 线圈电流 | Coil Current | I_coil |
| RMS电流 | RMS Current | I_rms |
| 正向压降 | Forward Voltage Drop | V_f |
| 峰值反向电压 | Peak Reverse Voltage | V_RRM |
| 漏源电压 | Drain-Source Voltage | V_DS |
| 漏极电流 | Drain Current | I_D |
| 导通电阻 | On-Resistance | R_DS(on) |
| 输出电容 | Output Capacitance | C_oss |
| 趋肤深度 | Skin Depth | δ |
| 谐振电容 | Resonant Capacitor | C₁, C₂ |
| 补偿电容 | Compensation Capacitor | Cf |
| 补偿电感 | Compensation Inductor | Lf |
| 互感 | Mutual Inductance | M |
| 系统效率 | System Efficiency | η |
| 传输效率 | Transfer Efficiency | η |

### 11.5 保护功能术语 (Protection Function Terms)

| 中文 | English | 说明 (Description) |
|------|---------|-------------------|
| 过流保护 | Overcurrent Protection | 电流超过阈值时保护 (Protection when current exceeds threshold) |
| 过温保护 | Overtemperature Protection | 温度超过阈值时保护 (Protection when temperature exceeds threshold) |
| 过充保护 | Overcharge Protection | 电池过充时保护 (Protection when battery is overcharged) |
| 过放保护 | Over-Discharge Protection | 电池过放时保护 (Protection when battery is over-discharged) |
| 保护阈值 | Protection Threshold | 触发保护的值 (Value that triggers protection) |

### 11.6 EMI/EMC术语 (EMI/EMC Terms)

| 中文 | English | 说明 (Description) |
|------|---------|-------------------|
| 电磁干扰 | Electromagnetic Interference | EMI |
| 电磁兼容 | Electromagnetic Compatibility | EMC |
| 辐射发射 | Radiated Emission | 通过空间传播的干扰 (Interference transmitted through space) |
| 传导发射 | Conducted Emission | 通过导线传播的干扰 (Interference transmitted through wires) |
| FCC认证 | FCC Certification | 美国联邦通信委员会认证 (US Federal Communications Commission certification) |
| CE认证 | CE Marking | 欧盟符合性认证 (European Conformity marking) |

---

*文档创建日期：2025年*
*目的：提供完整的电路实现和测试指南，确保无线充电方案可靠工作*

