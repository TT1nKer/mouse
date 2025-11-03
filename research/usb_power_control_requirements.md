# USB功率控制要求
## USB Power Control Requirements for Resonant Coupling Mouse Charger

---

## 一、问题概述 (Problem Overview)

### 1.1 为什么需要功率控制？(Why Power Control is Needed?)

**关键约束**:
1. **USB功率限制** (USB Power Limits):
   - USB 2.0: 5V @ 500mA = **2.5W 最大**
   - USB 3.0: 5V @ 900mA = **4.5W 最大**
   - 标准USB端口通常只提供 500mA

2. **设计目标** (Design Target):
   - 输入功率：≤ 1W（安全余量）
   - 输出功率：100 mW（维持鼠标使用）
   - 系统效率：> 70%

3. **谐振耦合特性** (Resonant Coupling Characteristics):
   - 如果不加控制，系统可能尝试传输更大功率
   - 需要限制输入电流，防止超过USB限制
   - 需要根据负载自动调整传输功率

---

## 二、功率控制方案 (Power Control Solutions)

### 2.1 发射端功率控制 (Transmitter Power Control)

#### 方案A：USB输入电流限制 (USB Input Current Limiting)

**实现方法**:

```
USB 5V ── [电流检测] ── [MCU控制] ── [Buck转换器] ── 12V ── [逆变器]
         ↑                                    ↑
         └─── 限制在200mA以内 (I_in < 200mA)
```

**电路实现**:
1. **USB输入电流检测**:
   ```
   使用分流电阻测量USB输入电流：
   I_USB = V_shunt / R_shunt
   
   目标：I_USB < 200mA (1W / 5V = 200mA)
   ```

2. **MCU控制逻辑**:
   ```c
   // 伪代码示例
   if (I_USB > 200mA) {
       // 降低PWM占空比或频率
       reduce_power_transmission();
   }
   
   if (I_USB < 150mA && efficiency_ok) {
       // 可以适当增加功率（如果效率良好）
       optimize_power_transmission();
   }
   ```

#### 方案B：PWM占空比控制 (PWM Duty Cycle Control)

**原理**:
- 通过调节逆变器PWM占空比来控制传输功率
- 降低占空比 → 降低平均功率
- 保持谐振频率不变

**实现**:
```
占空比控制：
D_min = 30% (最小传输功率)
D_nominal = 50% (标称功率)
D_max = 70% (最大功率，但受USB限制)

实际功率：P_actual = P_max × (D_actual / D_max)
```

#### 方案C：频率微调控制 (Frequency Tuning Control)

**原理**:
- 略微偏离谐振频率可以降低功率传输
- 但会降低效率，不推荐作为主要控制方法

**建议**: 仅在紧急情况下使用（如过载保护）

---

### 2.2 负载检测与匹配 (Load Detection and Matching)

#### 接收端存在检测 (Receiver Presence Detection)

**方法1：阻抗检测** (Impedance Detection):
```
- 检测发射端输入阻抗变化
- 无负载时：高阻抗
- 有负载时：阻抗下降
- MCU判断接收端是否存在
```

**方法2：反射功率检测** (Reflected Power Detection):
```
- 测量前向功率和反射功率
- 无接收端：大部分功率反射
- 有接收端：反射功率下降
```

**方法3：接收端信标信号** (Receiver Beacon Signal) ⭐推荐:
```
- 接收端发送低功率信标（通过负载调制）
- 发射端检测信标存在
- 只有检测到信标才启动大功率传输
```

#### 负载自适应 (Load Adaptation)

**Qi标准类似方法**:
1. **轻载检测** (Light Load Detection):
   - 如果接收端不吸收功率，降低传输功率
   - 进入待机模式，降低功耗

2. **重载检测** (Heavy Load Detection):
   - 如果负载需求增加，逐渐增加功率
   - 但仍受USB电流限制约束

---

### 2.3 完整功率控制框图 (Complete Power Control Block Diagram)

```
┌─────────────────────────────────────────────────────┐
│           发射端功率控制系统                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  USB 5V ──┬── [电流检测] ──┬── [MCU ADC]           │
│           │                │                       │
│           │                └── [功率限制控制]      │
│           │                                     │   │
│           └── [Buck转换器] ←── [PWM控制] ←──────┘   │
│                             ↑                       │
│                             │                       │
│  [接收端检测] ──→ [MCU] ────┘                       │
│                             │                       │
│  [线圈电流检测] ──→ [MCU ADC]                       │
│                             │                       │
│  [温度检测] ──→ [MCU ADC] ──┘                       │
│                                                     │
│  [MCU] ──→ [PWM生成] ──→ [逆变器驱动] ──→ [线圈]   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 三、具体实现方案 (Implementation Details)

### 3.1 USB输入电流限制 (USB Input Current Limiting)

**硬件电路**:

```
USB_VCC (5V) ──┬── [R_shunt (0.1Ω)] ──┬── VCC (给系统供电)
               │                       │
               │                    [滤波电容]
               │                       │
               └── [差分放大器] ──→ MCU_ADC
```

**参数选择**:
- **分流电阻** R_shunt = 0.1Ω（1%精度）
- **差分放大器** 增益 = 10
- **电压范围**: V_sense = I_USB × 0.1Ω × 10 = I_USB × 1V/A
  - 200mA → 0.2V
  - 300mA → 0.3V
  - MCU ADC可以精确测量

**软件实现**:
```c
// 示例代码（伪代码）
#define USB_CURRENT_MAX_MA  200  // 最大200mA
#define USB_CURRENT_NOMINAL_MA 150  // 标称150mA

uint16_t read_usb_current(void) {
    uint16_t adc_value = read_adc(USB_CURRENT_ADC_CH);
    // 假设ADC: 0-4095对应0-3.3V
    float voltage = (adc_value * 3.3) / 4095.0;
    float current_ma = (voltage / 1.0) * 1000.0;  // V_sense = I × 1V/A
    return (uint16_t)current_ma;
}

void power_control_loop(void) {
    uint16_t i_usb = read_usb_current();
    
    if (i_usb > USB_CURRENT_MAX_MA) {
        // 超过限制，降低功率
        pwm_duty_cycle -= 5;  // 降低占空比
        if (pwm_duty_cycle < 30) pwm_duty_cycle = 30;  // 最小30%
        
    } else if (i_usb < USB_CURRENT_NOMINAL_MA) {
        // 低于标称值，可以优化功率
        if (receiver_detected && efficiency_ok) {
            pwm_duty_cycle += 2;  // 稍微增加
            if (pwm_duty_cycle > 60) pwm_duty_cycle = 60;  // 最大60%
        }
    }
    
    update_pwm_duty_cycle(pwm_duty_cycle);
}
```

---

### 3.2 接收端检测 (Receiver Detection)

#### 方法：负载调制检测 (Load Modulation Detection)

**原理**:
- 接收端通过开关负载来调制反射阻抗
- 发射端检测阻抗变化
- 类似Qi标准的通信协议

**实现**:
```c
// 检测接收端是否存在
bool detect_receiver(void) {
    // 1. 发送低功率探测信号
    set_pwm_duty_cycle(20);  // 20%占空比，低功率
    
    // 2. 检测线圈阻抗变化
    float impedance = measure_coil_impedance();
    
    // 3. 判断是否有接收端
    if (impedance < Z_no_load_threshold) {
        return true;  // 检测到接收端
    }
    
    return false;
}

// 主循环中的检测
void main_loop(void) {
    static bool receiver_connected = false;
    
    // 定期检测接收端
    if (detect_receiver()) {
        if (!receiver_connected) {
            receiver_connected = true;
            // 启动正常功率传输
            set_pwm_duty_cycle(50);  // 50%占空比
            enable_power_transmission();
        }
    } else {
        if (receiver_connected) {
            receiver_connected = false;
            // 进入待机模式，降低功耗
            set_pwm_duty_cycle(10);  // 10%占空比，待机
        }
    }
}
```

---

### 3.3 自适应功率调整 (Adaptive Power Adjustment)

**控制策略**:

```
状态机：
┌─────────┐    检测到接收端     ┌──────────────┐
│  待机   │ ───────────────→ │  正常传输     │
│ (10%)   │                   │  (50%)       │
└─────────┘ ←────────────────  └──────────────┘
    ↑             接收端断开         │
    │                               │
    └─────────── USB电流超限 ←──────┘
                 (降低功率)
```

**实现代码**:
```c
typedef enum {
    STATE_STANDBY,      // 待机：无接收端
    STATE_LOW_POWER,    // 低功率：USB电流接近限制
    STATE_NORMAL,       // 正常：正常功率传输
    STATE_PROTECTION    // 保护：超过限制，关闭
} power_state_t;

power_state_t current_state = STATE_STANDBY;

void power_state_machine(void) {
    uint16_t i_usb = read_usb_current();
    bool receiver = detect_receiver();
    
    switch (current_state) {
        case STATE_STANDBY:
            if (receiver) {
                current_state = STATE_NORMAL;
                set_pwm_duty_cycle(50);
            }
            break;
            
        case STATE_NORMAL:
            if (!receiver) {
                current_state = STATE_STANDBY;
                set_pwm_duty_cycle(10);
            } else if (i_usb > 180) {  // 接近限制
                current_state = STATE_LOW_POWER;
                set_pwm_duty_cycle(40);
            }
            break;
            
        case STATE_LOW_POWER:
            if (!receiver) {
                current_state = STATE_STANDBY;
                set_pwm_duty_cycle(10);
            } else if (i_usb < 150) {
                current_state = STATE_NORMAL;
                set_pwm_duty_cycle(50);
            } else if (i_usb > 200) {
                current_state = STATE_PROTECTION;
                disable_power_transmission();
            }
            break;
            
        case STATE_PROTECTION:
            // 等待电流下降后恢复
            if (i_usb < 100 && receiver) {
                current_state = STATE_LOW_POWER;
                enable_power_transmission();
                set_pwm_duty_cycle(30);
            }
            break;
    }
}
```

---

## 四、功率控制参数设置 (Power Control Parameters)

### 4.1 推荐参数 (Recommended Parameters)

| 参数 | 值 | 说明 |
|------|-----|------|
| USB最大电流 | 200 mA | 安全限制（1W / 5V） |
| USB标称电流 | 150 mA | 正常工作时目标值 |
| 待机功率 | < 50 mW | 无接收端时功耗 |
| 正常传输功率 | 50% 占空比 | 接收端连接时 |
| 低功率模式 | 40% 占空比 | USB电流接近限制时 |
| 最小占空比 | 10% | 待机模式 |
| 最大占空比 | 60% | 不超过USB限制 |

### 4.2 功率分配 (Power Budget)

```
USB输入：1W (5V × 200mA)
├── MCU控制电路：50 mW
├── Buck转换器效率损失：~10% → 900 mW可用
├── 逆变器效率损失：~10% → 810 mW可用
├── 发射线圈效率损失：~10% → 730 mW可用
└── 传输到接收端：730 mW
    ├── 接收线圈效率：~95% → 693 mW
    ├── 整流效率：~90% → 624 mW
    ├── DC-DC效率：~85% → 530 mW
    └── 最终输出：> 500 mW (足够100mW需求 + 充电)
```

**结论**: 即使在70%系统效率下，仍有足够余量。

---

## 五、保护功能 (Protection Features)

### 5.1 USB过流保护 (USB Overcurrent Protection)

**实现**:
```c
#define USB_CURRENT_TRIP  220  // 跳闸点：220mA
#define USB_CURRENT_RESET 180  // 复位点：180mA

void usb_overcurrent_protection(void) {
    uint16_t i_usb = read_usb_current();
    
    if (i_usb > USB_CURRENT_TRIP) {
        // 立即关闭功率传输
        disable_power_transmission();
        set_led_error();  // 错误指示
        
        // 等待电流下降
        while (i_usb > USB_CURRENT_RESET) {
            k_sleep(K_MSEC(100));
            i_usb = read_usb_current();
        }
        
        // 恢复（低功率启动）
        enable_power_transmission();
        set_pwm_duty_cycle(30);  // 低功率重启
    }
}
```

### 5.2 功率限制优先级 (Power Limiting Priority)

```
优先级（从高到低）：
1. USB电流限制（硬件限制，必须遵守）
2. 温度保护（安全）
3. 过流保护（保护电路）
4. 负载匹配优化（性能）
```

---

## 六、测试要点 (Testing Points)

### 6.1 必须测试的项目 (Must-Test Items)

1. **USB电流限制测试**:
   - [ ] 验证输入电流不超过200mA
   - [ ] 测试过流保护功能
   - [ ] 测试不同USB端口的兼容性

2. **功率控制响应**:
   - [ ] 测试接收端连接/断开时的功率变化
   - [ ] 测试负载变化时的自适应调整
   - [ ] 测试从待机到正常传输的启动时间

3. **效率验证**:
   - [ ] 在功率控制下验证系统效率 > 70%
   - [ ] 测试不同功率点的效率
   - [ ] 验证功率控制不影响效率

---

## 七、总结 (Summary)

### 7.1 关键要点 (Key Points)

✅ **必须实现的功率控制**:
1. USB输入电流限制（硬件 + 软件）
2. 接收端检测（避免空载损耗）
3. 自适应功率调整（根据负载和USB限制）
4. 过流保护（安全）

✅ **功率控制的好处**:
- 遵守USB规范，避免损坏电脑USB端口
- 提高系统效率（只在需要时传输功率）
- 延长系统寿命（降低热损耗）
- 保护功能（安全）

✅ **实现复杂度**:
- 需要MCU实现控制逻辑
- 需要电流检测电路
- 需要接收端检测机制
- **但这些都是必需的！**

---

## 八、推荐方案 (Recommended Solution)

### 完整功率控制系统

**硬件**:
1. USB输入电流检测电路（分流电阻 + 差分放大器）
2. MCU（如STM32F0或ESP32，低成本）
3. PWM控制器（MCU内置或外部）
4. 接收端检测电路（阻抗检测）

**软件**:
1. 电流监测循环
2. 接收端检测算法
3. 功率状态机
4. 保护功能

**成本增加**: 
- 硬件：~$2-5（MCU + 检测电路）
- 软件：已包含在开发中
- **这是必需的功能，不能省略！**

---

*文档创建日期：2025年*
*目的：说明USB功率控制的必要性和实现方案*

