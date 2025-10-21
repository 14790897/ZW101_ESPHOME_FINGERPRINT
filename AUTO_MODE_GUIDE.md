# ZW101 自动模式 (Auto Mode) 详细说明

## 概述

**Auto Mode（自动模式）** 是 ZW101 指纹模组的一种工作模式，让**模组内部自动处理**指纹注册或匹配流程，无需 ESP32 持续发送命令。

---

## 自动模式的类型

ZW101 支持两种自动模式：

### 1️⃣ **自动注册模式 (Auto Enroll Mode)**

**指令码**: `0x31` (CMD_AUTO_ENROLL)

**功能**: 模组自动完成指纹注册的所有步骤

**工作流程**:
```
启动自动注册模式
   ↓
模组自动等待手指
   ↓
检测到手指 → 自动采集第1次
   ↓
等待手指移开
   ↓
等待手指再次放置 → 采集第2次
   ↓
... 重复 ...
   ↓
采集完5次 → 自动合并特征
   ↓
自动存储到指纹库
   ↓
完成 / 超时
```

**优势**:
- ✅ ESP32 只需发送一次命令
- ✅ 模组内部处理所有细节
- ✅ 减少 UART 通信量
- ✅ 更可靠（模组固件优化）

---

### 2️⃣ **自动匹配模式 (Auto Match Mode)**

**指令码**: `0x32` (CMD_AUTO_MATCH)

**功能**: 模组自动持续匹配指纹

**工作流程**:
```
启动自动匹配模式
   ↓
模组持续监听手指
   ↓
检测到手指 → 自动采集图像
   ↓
自动生成特征
   ↓
自动搜索指纹库
   ↓
找到匹配 → 发送结果给ESP32
   ↓
继续监听下一个手指
   ↓
(循环直到被取消)
```

**优势**:
- ✅ ESP32 不需要每秒发送搜索命令
- ✅ 模组自主工作
- ✅ 降低 ESP32 CPU 负载
- ✅ 可能支持模组内部的 LED 控制

---

## 代码实现

### 变量定义 (`zw101.h:125-126`)

```cpp
// 休眠和自动模式状态
bool auto_mode_active_{false};   // 是否处于自动模式
uint32_t auto_mode_timeout_{0};  // 自动模式超时时间（毫秒）
```

---

### 自动注册模式函数

**函数签名** (`zw101.cpp:522-554`):
```cpp
bool auto_enroll_mode(uint16_t timeout_sec = 60);
```

**参数**:
- `timeout_sec`: 超时时间（秒），默认 60 秒
  - 如果在超时内未完成注册，自动取消

**数据包格式**:
```
EF 01 FF FF FF FF 01 00 06 31 [超时高] [超时低] 00 [校验和]
│  │  └─────┬────┘ │  └──┬─┘ │  └──────┬─────┘ │  └───┬──┘
│  │     设备地址    │   长度  │    超时(ms)    保留  校验和
│  │               包标识    指令0x31
包头
```

**示例**:
```cpp
// 60秒超时的自动注册
id(zw101_reader).auto_enroll_mode(60);

// 5分钟超时
id(zw101_reader).auto_enroll_mode(300);
```

**状态变化**:
```cpp
auto_mode_active_ = true;
auto_mode_timeout_ = millis() + (timeout_sec * 1000);
status_sensor_ = "Auto Enroll Mode";
```

---

### 自动匹配模式函数

**函数签名** (`zw101.cpp:557-596`):
```cpp
bool auto_match_mode();
```

**参数**: 无超时（持续运行直到取消）

**数据包格式**:
```
EF 01 FF FF FF FF 01 00 08 32 02 [起始页] [页数] [安全级别] [校验和]
│  │  └─────┬────┘ │  └──┬─┘ │  │  └──┬─┘ └─┬─┘ │        └───┬──┘
│  │     设备地址    │   长度  │  │   起始   页数  安全级别    校验和
│  │               包标识    │  Buffer ID
包头                      指令0x32
```

**参数说明**:
- `buffer_id`: 2 (使用 Buffer 2 存储临时特征)
- `start_page`: 0 (从第一个指纹开始搜索)
- `page_num`: 50 (搜索所有指纹，ID 0-49)
- `security_level`: 2 (安全等级，值越高越严格)

**示例**:
```cpp
// 启动自动匹配
id(zw101_reader).auto_match_mode();

// 模组会持续监听并匹配指纹，直到调用:
id(zw101_reader).cancel_auto_mode();
```

**状态变化**:
```cpp
auto_mode_active_ = true;
auto_mode_timeout_ = 0;  // 无超时
status_sensor_ = "Auto Match Mode";
```

---

### 取消自动模式

**函数签名** (`zw101.cpp:601-625`):
```cpp
void cancel_auto_mode();
```

**指令码**: `0x30` (CMD_AUTO_CANCEL)

**数据包格式**:
```
EF 01 FF FF FF FF 01 00 03 30 00 34
│  │  └─────┬────┘ │  └──┬─┘ │  └─┬─┘
│  │     设备地址    │   长度  │  校验和
│  │               包标识   指令0x30
包头
```

**示例**:
```cpp
// 取消任何自动模式（注册或匹配）
id(zw101_reader).cancel_auto_mode();
```

**状态变化**:
```cpp
auto_mode_active_ = false;
auto_mode_timeout_ = 0;
status_sensor_ = "Auto Mode Cancelled";
```

---

## Auto Mode 对系统的影响

### 当 `auto_mode_active_ = true` 时：

#### ✅ **会发生的事情**:

1. **ESP32 停止主动搜索**
   ```cpp
   // zw101.cpp:55-58
   if (auto_mode_active_ || sleep_mode_) {
       return;  // 跳过 process_search()
   }
   ```

2. **自动检查超时**
   ```cpp
   // zw101.cpp:36-40
   if (auto_mode_active_ && auto_mode_timeout_ > 0 && now >= auto_mode_timeout_) {
       cancel_auto_mode();  // 超时自动取消
   }
   ```

3. **模组自主工作**
   - 自动检测手指
   - 自动处理指纹数据
   - 完成后通过 UART 发送结果

#### ❌ **不会发生的事情**:

1. **ESP32 不再发送搜索命令** - `process_search()` 被跳过
2. **ESP32 不发送图像采集命令** - 由模组内部处理
3. **无法手动触发单次搜索** - 除非先取消自动模式

---

## 使用场景

### 场景1: 简化指纹注册

**传统方式**（当前实现）:
```
ESP32 循环执行:
1. 等待手指 → 发送 GET_IMAGE
2. 生成特征 → 发送 GEN_CHAR
3. 等待移开 → 延时
4. 重复5次
5. 合并特征 → 发送 REG_MODEL
6. 存储 → 发送 STORE_CHAR
```

**自动注册方式**:
```
ESP32 一次性:
1. 发送 AUTO_ENROLL 命令
2. 等待完成通知
✅ 完成！
```

---

### 场景2: 降低 ESP32 负载

**传统方式**:
```cpp
void loop() {
    process_search();  // 每次 loop 都要处理状态机
}
// CPU 占用: 中等
```

**自动匹配方式**:
```cpp
void setup() {
    auto_match_mode();  // 只发送一次命令
}

void loop() {
    // process_search() 被跳过
    // 其他任务...
}
// CPU 占用: 低
```

---

### 场景3: LED 控制

**问题**: 你的 LED 设置会被 `process_search()` 干扰

**解决方案A**: 使用休眠模式（我们已实现）
```cpp
disable_auto_search();  // sleep_mode_ = true
```

**解决方案B**: 使用自动匹配模式
```cpp
auto_match_mode();  // auto_mode_active_ = true
```

**对比**:
| 方式 | ESP32搜索 | 模组工作 | LED持久性 | 指纹识别 |
|------|----------|---------|----------|---------|
| disable_auto_search | ❌ 停止 | ❌ 停止 | ✅ 持久 | ❌ 不可用 |
| auto_match_mode | ❌ 停止 | ✅ 自动 | ✅ 持久 | ✅ 可用 |

**推荐**: 如果需要同时保持 LED 和指纹识别，使用 `auto_match_mode()`！

---

## 配置文件中的使用

### 当前配置 (`fingerprint-zw101-new.yaml`)

#### 按钮定义:

```yaml
# 自动注册
- platform: template
  name: "ZW101 Fingerprint Auto Enroll"
  id: auto_enroll_button
  on_press:
    - lambda: |-
        int timeout = (int)id(enroll_timeout).state;
        id(zw101_reader).auto_enroll_mode(timeout);

# 自动匹配
- platform: template
  name: "ZW101 Fingerprint Auto Match"
  id: auto_match_button
  on_press:
    - lambda: |-
        id(zw101_reader).auto_match_mode();

# 取消自动模式
- platform: template
  name: "ZW101 Fingerprint Cancel Auto"
  id: cancel_auto_button
  on_press:
    - lambda: |-
        id(zw101_reader).cancel_auto_mode();
```

#### API 服务:

```yaml
api:
  services:
    # 自动注册
    - service: auto_enroll
      variables:
        timeout: int
      then:
        - lambda: |-
            id(zw101_reader).auto_enroll_mode(timeout);

    # 自动匹配
    - service: auto_match
      then:
        - lambda: |-
            id(zw101_reader).auto_match_mode();

    # 取消自动模式
    - service: cancel_auto
      then:
        - lambda: |-
            id(zw101_reader).cancel_auto_mode();
```

---

## 实际测试

### 测试1: 自动注册

```yaml
# Home Assistant 中调用
service: esphome.fingerprint_zw101_auto_enroll
data:
  timeout: 120  # 2分钟超时
```

**预期行为**:
1. 模组开始监听手指
2. 放置手指 → 听到提示音
3. 移开手指
4. 再次放置... 重复5次
5. 注册完成或超时

**日志输出**:
```
[I][zw101]: Auto enroll mode activated, timeout: 120 seconds
[I][zw101]: Auto Enroll Mode
... (等待模组完成) ...
[I][zw101]: Auto mode timeout, cancelling  (如果超时)
```

---

### 测试2: 自动匹配（推荐用于 LED 持久化）

```yaml
button:
  - platform: template
    name: "Enable Auto Match and Set LED"
    on_press:
      - lambda: |-
          // 设置 LED
          id(zw101_reader).set_rgb_led(3, 4, 150);  // 红色常亮
          delay(100);

          // 启用自动匹配（模组自己处理识别）
          id(zw101_reader).auto_match_mode();

          ESP_LOGI("main", "LED set, auto match enabled");
```

**优势**:
- ✅ LED 持久保持（ESP32 不再发送搜索命令）
- ✅ 指纹识别仍然有效（由模组内部处理）
- ✅ 降低 ESP32 CPU 占用

---

## 对比：三种工作模式

| 模式 | ESP32 任务 | 模组任务 | LED 控制 | 功耗 | 适用场景 |
|------|-----------|---------|---------|------|---------|
| **正常搜索** | 每秒发送搜索命令 | 响应命令 | ❌ 会被干扰 | 高 | 默认模式 |
| **自动模式** | 只发送一次命令 | 自主工作 | ✅ 持久 | 中 | 简化控制 |
| **休眠模式** | 停止搜索 | 低功耗待机 | ✅ 持久 | 低 | 省电/装饰 |

---

## 注意事项

### ⚠️ **已知限制**

1. **同时只能有一个自动模式**
   ```cpp
   if (auto_mode_active_) {
       ESP_LOGW(TAG, "Auto mode already active");
       return false;
   }
   ```

2. **自动模式期间无法手动控制**
   - 不能手动触发搜索
   - 不能手动注册
   - 必须先取消自动模式

3. **响应数据格式可能不同**
   - 自动模式的响应格式需要单独解析
   - 当前实现可能未完全处理响应

---

## 改进建议

### 建议1: 添加自动模式响应处理

```cpp
// 在 loop() 中添加自动模式响应处理
if (auto_mode_active_ && available()) {
    // 解析自动模式的响应
    // 可能包含：注册成功、匹配成功、超时等
}
```

### 建议2: 用自动匹配替代 process_search

```cpp
// zw101.cpp setup()
void ZW101Component::setup() {
    // ...

    // 直接使用自动匹配替代循环搜索
    auto_match_mode();
}
```

**优势**:
- 简化代码
- 降低功耗
- LED 不会被干扰

---

## 总结

### Auto Mode 的本质

**Auto Mode = 把控制权交给模组内部固件**

- ✅ ESP32 只需发送一次命令
- ✅ 模组自主完成复杂流程
- ✅ 减少 UART 通信
- ✅ 可能更可靠（厂商优化）

### 何时使用

| 需求 | 推荐模式 |
|------|---------|
| 简化注册流程 | `auto_enroll_mode()` |
| 降低 CPU 占用 | `auto_match_mode()` |
| LED 持久化 + 识别 | `auto_match_mode()` ⭐ |
| 纯 LED 装饰 | `disable_auto_search()` |
| 省电 | `enter_sleep_mode()` |

---

**版本**: 1.0
**最后更新**: 2025
**适用**: ZW101 ESPHome Component
