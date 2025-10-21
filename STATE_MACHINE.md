# ZW101 指纹模块状态机技术文档

## 目录
- [概述](#概述)
- [搜索状态机 (Search State Machine)](#搜索状态机-search-state-machine)
- [注册状态机 (Enroll State Machine)](#注册状态机-enroll-state-machine)
- [时序图](#时序图)
- [配置参数](#配置参数)
- [故障排查](#故障排查)

---

## 概述

ZW101 指纹模块使用**非阻塞式状态机**设计，确保不会阻塞 ESP32 的主循环，避免看门狗超时。模块包含两个独立的状态机：

1. **搜索状态机** (`search_state_`) - 负责自动搜索和匹配指纹
2. **注册状态机** (`enroll_state_`) - 负责新指纹的注册流程

### 核心特性
- ✅ 非阻塞设计，所有操作异步执行
- ✅ 自动重试机制，提高识别成功率
- ✅ 状态隔离，注册和搜索互不干扰
- ✅ 完整的错误处理和超时保护

---

## 搜索状态机 (Search State Machine)

### 状态定义

```cpp
// 文件: zw101.h:95-101
enum SearchState {
  SEARCH_IDLE,        // 空闲状态，等待开始新的搜索
  SEARCH_GET_IMAGE,   // 正在获取指纹图像
  SEARCH_GEN_CHAR,    // 正在生成指纹特征
  SEARCH_WAIT_RETRY,  // 等待重试
  SEARCH_DO_SEARCH    // 执行指纹搜索匹配
};
```

### 状态流转图

```
                    启动后每1秒触发
                    ┌──────────────┐
                    │              │
                    ▼              │
        ┌───────────────────────┐  │
        │   SEARCH_IDLE         │  │
        │   空闲状态             │  │
        └───────────┬───────────┘  │
                    │               │
                    │ 1000ms 定时器触发
                    ▼               │
        ┌───────────────────────┐  │
        │ SEARCH_GET_IMAGE      │  │
        │ 获取指纹图像           │  │
        │ CMD: 0x01             │  │
        └───────┬───────┬───────┘  │
                │       │           │
         检测到指纹  无指纹         │
                │       │           │
                ▼       ▼           │
         ┌──────────┐ ┌─────────┐  │
         │GEN_CHAR  │ │WAIT     │  │
         │生成特征   │ │_RETRY   │  │
         │CMD: 0x02 │ │等待500ms│  │
         └────┬─────┘ └────┬────┘  │
              │            │        │
         成功/失败         │        │
              │            └────────┘
              ▼                (重新获取图像)
    ┌─────────┴──────────┐
    │ 成功              失败
    ▼                    ▼
┌────────────┐     ┌──────────┐
│DO_SEARCH   │     │重试次数<5?│
│执行搜索     │     └─────┬────┘
│CMD: 0x04   │           │
│范围: 0-49  │      Yes  │  No
└─────┬──────┘           │  │
      │              ┌───┘  └──┐
      │              ▼          ▼
      │        WAIT_RETRY    IDLE
      │        (等待重试)   (放弃)
      │              │
      └──────────────┴─────────┐
                                │
                    完成后返回空闲
                                │
                                ▼
                        [ SEARCH_IDLE ]
```

### 详细状态说明

#### 1. SEARCH_IDLE (空闲状态)
**代码位置**: `zw101.cpp:69-76`

**功能**: 等待状态，每1秒自动触发一次新的搜索流程

**触发条件**:
```cpp
if (now - search_last_action_ > 1000) {
    search_state_ = SEARCH_GET_IMAGE;
    search_retry_count_ = 0;
    search_last_action_ = now;
}
```

**关键变量**:
- `search_last_action_`: 上次动作的时间戳
- `search_retry_count_`: 重试计数器（重置为0）

**下一状态**: `SEARCH_GET_IMAGE`

---

#### 2. SEARCH_GET_IMAGE (获取图像)
**代码位置**: `zw101.cpp:78-89`

**功能**: 检测并获取指纹图像

**执行命令**:
```cpp
send_cmd(CMD_GET_IMAGE);  // 指令码: 0x01
```

**状态转换**:
| 条件 | 下一状态 | 说明 |
|------|---------|------|
| `receive_response() == true` | `SEARCH_GEN_CHAR` | 检测到指纹，继续处理 |
| `receive_response() == false` | `SEARCH_WAIT_RETRY` | 未检测到指纹，等待重试 |

**协议数据包**:
```
EF 01 FF FF FF FF 01 00 03 01 00 05
│  │  └─────┬────┘ │  └──┬─┘ │  └─┬─┘
│  │     设备地址    │   长度  │  校验和
│  │               包标识   指令码
包头
```

---

#### 3. SEARCH_GEN_CHAR (生成特征)
**代码位置**: `zw101.cpp:91-112`

**功能**: 将获取的指纹图像生成特征码存入 Buffer 1

**执行命令**:
```cpp
send_cmd2(CMD_GEN_CHAR, 1);  // 指令码: 0x02, 参数: Buffer ID = 1
```

**状态转换**:
| 条件 | 下一状态 | 说明 |
|------|---------|------|
| 成功 | `SEARCH_DO_SEARCH` | 特征生成成功，执行搜索 |
| 失败 + 重试<5次 | `SEARCH_WAIT_RETRY` | 重试 |
| 失败 + 重试≥5次 | `SEARCH_IDLE` | 放弃本次搜索 |

**重试逻辑**:
```cpp
search_retry_count_++;
if (search_retry_count_ >= 5) {
    status_sensor_->publish_state("No Valid Fingerprint");
    search_state_ = SEARCH_IDLE;
} else {
    search_state_ = SEARCH_WAIT_RETRY;
}
```

---

#### 4. SEARCH_WAIT_RETRY (等待重试)
**代码位置**: `zw101.cpp:114-119`

**功能**: 等待 500ms 后重新尝试获取图像

**等待时间**: 500 毫秒

**触发条件**:
```cpp
if (now - search_last_action_ > 500) {
    search_state_ = SEARCH_GET_IMAGE;
}
```

**用途**:
- 给用户时间调整手指位置
- 避免频繁发送指令导致模块忙碌
- 降低功耗

**下一状态**: `SEARCH_GET_IMAGE`

---

#### 5. SEARCH_DO_SEARCH (执行搜索)
**代码位置**: `zw101.cpp:121-152`

**功能**: 在指纹库中搜索匹配的指纹

**执行命令**:
```cpp
send_search_cmd(
    1,                    // Buffer ID = 1
    0,                    // 起始页: 0
    library_capacity_     // 搜索范围: 50 (0-49)
);
```

**搜索参数**:
- **Buffer ID**: 1 (存放当前指纹特征的缓冲区)
- **起始页**: 0 (从第一个指纹开始)
- **页数**: 50 (搜索 ID 0-49 的所有指纹)

**响应处理**:
```cpp
if (length >= 12 && response[9] == 0x00) {  // 确认码 0x00 = 成功
    uint16_t match_page = (response[10] << 8) | response[11];   // 匹配的ID
    uint16_t match_score = (response[12] << 8) | response[13];  // 匹配得分

    // 发布传感器数据
    fingerprint_sensor_->publish_state(true);      // 匹配状态: true
    match_id_sensor_->publish_state(match_page);   // 匹配ID
    match_score_sensor_->publish_state(match_score); // 得分
    status_sensor_->publish_state("Match Found");  // 状态文本

    // 3秒后自动清除匹配标志
    match_found_ = true;
    match_clear_time_ = now + 3000;
}
```

**协议数据包**:
```
搜索命令:
EF 01 FF FF FF FF 01 00 08 04 01 00 00 00 32 00 40
│  │  └─────┬────┘ │  └──┬─┘ │  │  └──┬─┘ └──┬─┘ └─┬─┘
│  │     设备地址    │   长度  │  │   起始页  页数  校验和
│  │               包标识   │  缓冲区ID
包头                      指令码0x04

搜索响应 (成功):
EF 01 FF FF FF FF 07 00 07 00 00 05 00 C8 00 D9
│  │  └─────┬────┘ │  └──┬─┘ │  └─┬─┘ └──┬─┘ └─┬─┘
│  │     设备地址    │   长度  │  匹配ID  得分  校验和
│  │              应答包    确认码0x00=成功
包头
```

**下一状态**: `SEARCH_IDLE` (无论成功或失败都返回空闲)

---

### 搜索流程时序图

```
时间轴 →

loop()  ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
调用    │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │
        └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘

状态    IDLE────────────►GET_IMAGE►GEN_CHAR►DO_SEARCH►IDLE─────
        │                    ▲                          │
        │   无指纹           │ 重试                     │
        │    ▼               │                          │
        └──►WAIT_RETRY───────┘                          │
        │      500ms                                    │
        │                                               │
        └──────────── 1000ms 后循环 ────────────────────┘

传感器          │          │        │         │
更新            │          │        │         └─► match_id = 5
                │          │        │         └─► match_score = 200
                │          │        │         └─► status = "Match Found"
                │          │        │         └─► fingerprint = true
                │          │        │
                │          │        └─► 特征生成 (Buffer 1)
                │          └─► 图像采集
                └─► 等待指纹
```

---

## 注册状态机 (Enroll State Machine)

### 状态定义

```cpp
// 文件: zw101.h:80-87
enum EnrollState {
  ENROLL_IDLE,          // 空闲，未进行注册
  ENROLL_WAIT_FINGER,   // 等待手指放置
  ENROLL_CAPTURING,     // 正在采集指纹图像
  ENROLL_WAIT_REMOVE,   // 等待手指移开
  ENROLL_MERGING,       // 合并特征模板
  ENROLL_STORING        // 存储模板到指纹库
};
```

### 状态流转图

```
                    用户触发注册
                         │
                         ▼
            ┌─────────────────────┐
            │   ENROLL_IDLE       │
            │   (空闲状态)         │
            └─────────────────────┘
                         │
                register_fingerprint()
                         │
                         ▼
            ┌─────────────────────┐
         ┌─►│ ENROLL_WAIT_FINGER  │◄─┐
         │  │ (等待手指)           │  │
         │  │ 检查间隔: 200ms      │  │
         │  └──────────┬──────────┘  │
         │             │              │
         │      检测到指纹            │
         │             │              │
         │             ▼              │
         │  ┌─────────────────────┐  │
         │  │ ENROLL_CAPTURING    │  │
         │  │ (采集指纹 N/5)       │  │
         │  │ CMD: GEN_CHAR       │  │
         │  └──────────┬──────────┘  │
         │             │              │
         │      采集成功?             │
         │       │    │ 失败          │
         │    成功   └───────────────┘
         │       │
         │       ▼
         │  样本数 < 5?
         │    │      │
         │   Yes     No
         │    │      │
         │    ▼      ▼
         │  ┌────┐ ┌──────────────┐
         │  │等移│ │ENROLL_MERGING│
         │  │开手│ │(合并5个样本)  │
         │  │指1s│ │CMD: REG_MODEL│
         │  └─┬──┘ └───────┬──────┘
         │    │            │
         └────┘      合并成功?
                      │    │
                    成功  失败
                      │    │
                      ▼    ▼
              ┌───────────┐ ┌────────┐
              │  STORING  │ │ IDLE   │
              │ (存储模板) │ │(失败)  │
              │CMD: STORE │ └────────┘
              └─────┬─────┘
                    │
              存储完成 (成功/失败)
                    │
                    ▼
              ┌──────────┐
              │  IDLE    │
              │ (完成)    │
              └──────────┘
```

### 详细状态说明

#### 1. ENROLL_IDLE (空闲状态)
**触发方式**:
- 用户在 Home Assistant 中打开 "Enroll" 开关
- 调用 API 服务 `auto_enroll`
- 按下配置中的 "Auto Enroll" 按钮

**代码位置**: `zw101.cpp:258-274`

**启动函数**:
```cpp
bool ZW101Component::register_fingerprint() {
    if (enroll_state_ != ENROLL_IDLE) {
        ESP_LOGW(TAG, "Enrollment already in progress");
        return false;
    }

    status_sensor_->publish_state("Enrolling...");
    ESP_LOGI(TAG, "Starting fingerprint enrollment");

    enroll_state_ = ENROLL_WAIT_FINGER;  // 转到等待手指状态
    enroll_sample_count_ = 0;            // 样本计数清零
    enroll_last_action_ = millis();

    return true;
}
```

---

#### 2. ENROLL_WAIT_FINGER (等待手指)
**代码位置**: `zw101.cpp:163-180`

**功能**: 每 200ms 检测一次是否有手指放置

**检测逻辑**:
```cpp
if (now - enroll_last_action_ > 200) {
    enroll_last_action_ = now;
    send_cmd(CMD_GET_IMAGE);  // 发送获取图像命令

    if (receive_response()) {
        // 检测到手指
        enroll_state_ = ENROLL_CAPTURING;
        ESP_LOGI(TAG, "Finger detected, capturing sample %d/5",
                 enroll_sample_count_ + 1);
    } else if (now - enroll_last_action_ > 30000) {
        // 超时30秒
        status_sensor_->publish_state("Enroll Timeout");
        enroll_state_ = ENROLL_IDLE;
    }
}
```

**超时保护**: 30 秒无响应自动取消

**下一状态**:
- 检测到手指 → `ENROLL_CAPTURING`
- 超时 → `ENROLL_IDLE`

---

#### 3. ENROLL_CAPTURING (采集指纹)
**代码位置**: `zw101.cpp:182-201`

**功能**: 生成指纹特征并存入对应的 Buffer

**执行命令**:
```cpp
send_cmd2(CMD_GEN_CHAR, enroll_sample_count_ + 1);
// 第1次: Buffer 1
// 第2次: Buffer 2
// 第3次: Buffer 3
// ...
// 第5次: Buffer 5
```

**采集流程**:
```cpp
if (receive_response()) {
    enroll_sample_count_++;
    ESP_LOGI(TAG, "Sample %d captured", enroll_sample_count_);

    if (enroll_sample_count_ >= 5) {
        // 收集完5个样本，进入合并阶段
        enroll_state_ = ENROLL_MERGING;
    } else {
        // 还需要更多样本，等待手指移开
        enroll_state_ = ENROLL_WAIT_REMOVE;
        enroll_last_action_ = now;
    }
} else {
    // 采集失败，返回等待手指状态
    enroll_state_ = ENROLL_WAIT_FINGER;
}
```

**样本要求**: 需要采集 **5 个样本**

**下一状态**:
- 样本 < 5 → `ENROLL_WAIT_REMOVE`
- 样本 = 5 → `ENROLL_MERGING`
- 失败 → `ENROLL_WAIT_FINGER`

---

#### 4. ENROLL_WAIT_REMOVE (等待移开手指)
**代码位置**: `zw101.cpp:203-210`

**功能**: 提示用户移开手指，准备下一次采集

**等待时间**: 1000ms

**逻辑**:
```cpp
if (now - enroll_last_action_ > 1000) {
    ESP_LOGI(TAG, "Remove finger and place again (%d/5)",
             enroll_sample_count_);
    enroll_state_ = ENROLL_WAIT_FINGER;
    enroll_last_action_ = now;
}
```

**用户提示**:
```
日志输出: "Remove finger and place again (2/5)"
状态传感器: "Enrolling... (2/5)"
```

**下一状态**: `ENROLL_WAIT_FINGER`

---

#### 5. ENROLL_MERGING (合并特征)
**代码位置**: `zw101.cpp:212-222`

**功能**: 将 5 个样本合并成一个完整的指纹模板

**执行命令**:
```cpp
send_cmd(CMD_REG_MODEL);  // 指令码: 0x05
```

**合并算法**:
- 由指纹模块内部处理
- 从 Buffer 1-5 中提取共同特征
- 生成一个稳定的模板

**状态转换**:
```cpp
if (receive_response()) {
    enroll_state_ = ENROLL_STORING;  // 成功，存储模板
} else {
    status_sensor_->publish_state("Enroll Failed - Merge");
    enroll_state_ = ENROLL_IDLE;     // 失败，返回空闲
}
```

**失败原因**:
- 5 个样本差异过大
- 指纹质量不佳
- 手指位置不一致

---

#### 6. ENROLL_STORING (存储模板)
**代码位置**: `zw101.cpp:224-247`

**功能**: 将合并后的模板存储到指纹库

**执行命令**:
```cpp
send_store_cmd(1, next_fingerprint_id_);
// 参数1: Buffer ID = 1 (合并后的模板)
// 参数2: 要存储的指纹ID
```

**ID 分配逻辑**:
```cpp
if (receive_response()) {
    ESP_LOGI(TAG, "Fingerprint enrolled successfully as ID %d",
             next_fingerprint_id_);

    // 更新状态传感器
    char buf[64];
    snprintf(buf, sizeof(buf), "Enroll Success (ID: %d)",
             next_fingerprint_id_);
    status_sensor_->publish_state(buf);

    // ID 自增，循环使用
    next_fingerprint_id_++;
    if (next_fingerprint_id_ >= library_capacity_) {  // library_capacity_ = 50
        next_fingerprint_id_ = 0;  // 超过容量从0开始
    }

    ESP_LOGI(TAG, "Next fingerprint will use ID: %d", next_fingerprint_id_);
}
```

**ID 范围**: 0-49 (共 50 个指纹位)

**存储失败处理**:
```cpp
else {
    status_sensor_->publish_state("Enroll Failed - Store");
}
enroll_state_ = ENROLL_IDLE;  // 无论成功失败都返回空闲
```

**下一状态**: `ENROLL_IDLE`

---

### 注册流程时序图

```
用户操作          │                                              │
                 ▼                                              ▼
Home Assistant   打开"Enroll"开关                              注册完成通知
                 │                                              │
                 │                                              │
ESP32           ┌┴─ register_fingerprint()                     │
                │                                               │
状态变化        IDLE ─► WAIT_FINGER ─► CAPTURING ─► WAIT_REMOVE
                         │  ▲            │             │
                         │  │200ms检测   │             │ 1000ms
                         │  └────────────┘             │
                         │                             ▼
                         │                          样本2/5
                         │                             │
                         │                             ▼
                         │                      WAIT_FINGER ─► CAPTURING
                         │                                        │
                         │                                      样本3/5
                         │                                        │
                         │                                        ▼
                         │                                   [重复直到5/5]
                         │                                        │
                         │                                        ▼
                         │                                    MERGING
                         │                                        │
                         │                                   合并5个样本
                         │                                        │
                         │                                        ▼
                         │                                    STORING
                         │                                        │
                         │                              存储到ID: next_id
                         │                                        │
                         └────────────────────────────────────► IDLE

UART通信        GET_IMAGE ───► GEN_CHAR(1) ───► ... ───► REG_MODEL ───► STORE
                  0x01           0x02                      0x05          0x06

时间线          0s ────► 1s ────► 5s ────► 10s ────► 15s ────► 16s ────► 完成
                │        │        │        │        │        │
                启动     样本1    样本2    样本3    样本4    样本5
                                                            合并+存储
```

---

## 时序图

### 完整工作流程

```
系统启动
  │
  ├─► setup()
  │    │
  │    └─► 初始化 search_state_ = SEARCH_IDLE
  │        初始化 enroll_state_ = ENROLL_IDLE
  │
  ├─► loop() [每次循环约10-20ms]
  │    │
  │    ├─► 0.5秒: 关闭LED
  │    ├─► 1秒: 读取模组信息 (read_fp_info)
  │    │         ├─► 读取容量: library_capacity_ = 50
  │    │         └─► 读取已注册数: next_fingerprint_id_ = N
  │    │
  │    ├─► 检查自动模式超时
  │    ├─► 处理匹配成功状态清除 (3秒后)
  │    │
  │    ├─► if (enroll_state_ != IDLE)
  │    │    └─► process_enrollment()  [优先级最高]
  │    │
  │    └─► if (!auto_mode && !sleep_mode)
  │         └─► process_search()      [后台持续运行]
  │              │
  │              └─► 每1秒自动搜索一次指纹
  │
  └─► yield() [让出CPU]
```

### 搜索与注册并发控制

```
时间线 ──────────────────────────────────────────►

搜索流程:
IDLE ─► GET ─► GEN ─► SEARCH ─► IDLE ─► GET ─► ...
                                  ▲
                                  │ 暂停搜索
                                  │
注册触发:                          │
用户操作 ────────────────────────► ENROLL_START
                                  │
                                  ▼
注册流程:                  WAIT_FINGER ─► CAPTURE ─► ... ─► STORE
                                                              │
                                                           完成注册
                                                              │
                                                              ▼
搜索恢复:                                               IDLE ─► GET ─► ...
```

**并发规则**:
```cpp
// zw101.cpp:50-53
if (enroll_state_ != ENROLL_IDLE) {
    process_enrollment();
    return;  // 注册过程中不进行自动搜索
}
```

---

## 配置参数

### 时间参数汇总

| 参数名称 | 默认值 | 位置 | 说明 |
|---------|--------|------|------|
| 搜索间隔 | 1000ms | `zw101.cpp:71` | 自动搜索的触发间隔 |
| 重试等待 | 500ms | `zw101.cpp:116` | 搜索失败后的等待时间 |
| 最大重试次数 | 5次 | `zw101.cpp:100` | 特征生成失败的最大重试 |
| 注册检测间隔 | 200ms | `zw101.cpp:165` | 注册时检测指纹的间隔 |
| 注册超时 | 30000ms | `zw101.cpp:173` | 等待手指放置的超时 |
| 移开手指等待 | 1000ms | `zw101.cpp:205` | 采集样本后等待移开 |
| 匹配清除延迟 | 3000ms | `zw101.cpp:146` | 匹配成功后状态保持时间 |
| LED 关闭延迟 | 500ms | `zw101.cpp:24` | 启动后关闭 LED 的延迟 |
| 信息读取延迟 | 1000ms | `zw101.cpp:31` | 启动后读取模组信息 |

### 容量参数

| 参数名称 | 值 | 位置 | 说明 |
|---------|---|------|------|
| 指纹库容量 | 50 | `zw101.h:92` | 最大可存储指纹数量 |
| 删除ID范围 | 0-49 | `fingerprint-zw101-new.yaml:118` | UI 中删除指纹的有效范围 |
| 注册样本数 | 5 | `zw101.cpp:189` | 每个指纹需要采集的样本数 |

### UART 参数

```yaml
# fingerprint-zw101-new.yaml:46-53
uart:
  id: fingerprint_uart
  tx_pin: GPIO1
  rx_pin: GPIO0
  baud_rate: 57600
  data_bits: 8
  parity: NONE
  stop_bits: 1
```

---

## 故障排查

### 搜索问题

#### 1. 一直显示 "No Valid Fingerprint"
**可能原因**:
- 指纹质量差（手指太干/太湿/有污渍）
- 传感器表面脏污
- 重试次数达到上限（5次）

**解决方案**:
```cpp
// 增加最大重试次数 (zw101.cpp:100)
if (search_retry_count_ >= 10) {  // 从5改为10
```

**诊断命令**:
```yaml
# 调用服务检查模组状态
service: esphome.fingerprint_zw101_check_online
```

---

#### 2. 搜索速度慢
**可能原因**:
- 搜索间隔设置过长
- 指纹库容量设置错误

**解决方案**:
```cpp
// 减少搜索间隔 (zw101.cpp:71)
if (now - search_last_action_ > 500) {  // 从1000改为500ms
```

**性能优化**:
```cpp
// 减少搜索范围（如果只使用了前10个ID）
send_search_cmd(1, 0, 10);  // 只搜索ID 0-9
```

---

#### 3. 匹配率低
**可能原因**:
- 注册时指纹位置不一致
- 搜索时手指位置偏移

**解决方案**:
1. 重新注册指纹，确保5次采样位置一致
2. 调整安全等级（降低匹配阈值）

```cpp
// zw101.cpp:556 - 修改安全等级
uint8_t security_level = 1;  // 从2改为1（降低要求）
```

---

### 注册问题

#### 1. 注册超时
**症状**: 显示 "Enroll Timeout"

**解决方案**:
```cpp
// 增加超时时间 (zw101.cpp:173)
} else if (now - enroll_last_action_ > 60000) {  // 从30秒改为60秒
```

---

#### 2. 合并失败 "Enroll Failed - Merge"
**可能原因**:
- 5次采样的指纹差异过大
- 手指位置不一致

**解决方案**:
1. 确保每次按压位置相同
2. 保持手指清洁干燥
3. 多尝试几次

---

#### 3. 存储失败 "Enroll Failed - Store"
**可能原因**:
- 指纹库已满（达到50个）
- 指定ID已被占用

**解决方案**:
```cpp
// 检查ID分配逻辑 (zw101.cpp:287-292)
next_fingerprint_id_ = 0;  // 清空后重置

// 或者先删除旧指纹
id(zw101_reader).delete_fingerprint(0);
```

---

### 通信问题

#### 1. 模组无响应
**诊断步骤**:
```cpp
// 1. 检查握手
id(zw101_reader).handshake();  // 应返回 true

// 2. 检查UART配置
ESP_LOGI(TAG, "UART TX: %d, RX: %d", tx_pin, rx_pin);

// 3. 测试波特率
// 尝试其他常见波特率: 9600, 19200, 38400, 57600, 115200
```

---

#### 2. 数据包校验错误
**症状**: 日志显示接收到数据但无法解析

**检查点**:
```cpp
// 验证数据包格式
uint8_t response[50];
uint8_t length = wait_for_response(response, 50, 500);

ESP_LOGI(TAG, "Response length: %d", length);
ESP_LOG_BUFFER_HEX(TAG, response, length);  // 打印原始数据

// 检查包头
if (response[0] == 0xEF && response[1] == 0x01) {
    ESP_LOGI(TAG, "Header OK");
}

// 检查确认码
if (response[9] == 0x00) {
    ESP_LOGI(TAG, "Command success");
} else {
    ESP_LOGW(TAG, "Error code: 0x%02X", response[9]);
}
```

---

### 性能优化

#### 1. 降低功耗
```cpp
// 在不使用时进入休眠模式
id(zw101_reader).enter_sleep_mode();

// 唤醒: 任何UART命令都会自动唤醒
```

#### 2. 减少日志输出
```yaml
# fingerprint-zw101-new.yaml:35-37
logger:
  level: INFO  # 从DEBUG改为INFO
  baud_rate: 115200
```

#### 3. 优化搜索策略
```cpp
// 只在特定条件下搜索（如检测到人体接近）
if (id(pir_sensor).state) {  // PIR 传感器检测到人
    process_search();
}
```

---

## 协议参考

### 常用指令码

| 指令码 | 名称 | 功能 | 参数 |
|-------|------|------|------|
| 0x01 | GET_IMAGE | 获取指纹图像 | 无 |
| 0x02 | GEN_CHAR | 生成特征 | Buffer ID (1-5) |
| 0x04 | SEARCH | 搜索指纹 | Buffer ID, 起始页, 页数 |
| 0x05 | REG_MODEL | 合并特征 | 无 |
| 0x06 | STORE_CHAR | 存储模板 | Buffer ID, 模板ID |
| 0x0C | DEL_CHAR | 删除模板 | 模板ID, 数量 |
| 0x0D | CLEAR_LIB | 清空指纹库 | 无 |
| 0x1D | READ_VALID_NUMS | 读有效模板数 | 无 |
| 0x31 | AUTO_ENROLL | 自动注册 | 超时时间(ms) |
| 0x32 | AUTO_MATCH | 自动匹配 | Buffer ID, 起始页, 页数 |
| 0x35 | HANDSHAKE | 握手测试 | 无 |
| 0x3C | RGB_CTRL | RGB灯控制 | 模式, 颜色, 亮度, 循环次数, 周期 |

### RGB LED 控制详细参数

**函数原型**:
```cpp
void set_rgb_led(uint8_t mode, uint8_t color, uint8_t brightness = 100);
```

**数据包格式**:
```
EF 01 FF FF FF FF 01 00 09 3C [模式] [颜色] [亮度] [循环] [周期] 00 [校验和]
│  │  └─────┬────┘ │  └──┬─┘ │  └───┬───┘ └──┬─┘ └──┬─┘ └─┬─┘ │  └───┬──┘
│  │     设备地址    │   长度  │    参数       亮度  循环  周期  保留  校验和
│  │               包标识    指令0x3C
包头
```

**参数说明**:

1. **模式 (mode)**:
   - 1 = 呼吸灯
   - 2 = 闪烁灯
   - 3 = 常亮
   - 4 = 关闭
   - 5 = 渐变开
   - 6 = 渐变关
   - 7 = 跑马灯

2. **颜色 (color)**:
   - 1 = 蓝色
   - 2 = 绿色
   - 3 = 青色
   - 4 = 红色
   - 5 = 紫色
   - 6 = 黄色
   - 7 = 白色

3. **亮度 (brightness)**:
   - 范围: 0-255
   - 默认: 100
   - 占空比控制

4. **循环次数 (loop_times)**:
   - 0xFF = 无限循环（推荐）
   - 0x00 = 默认次数（可能只执行几秒）
   - 其他 = 指定次数

5. **周期 (cycle)**:
   - 单位: 100ms
   - 默认: 15 (1.5秒)
   - 控制呼吸/闪烁的速度

**使用示例**:
```cpp
// 红色呼吸灯，持续运行
id(zw101_reader).set_rgb_led(1, 4, 150);  // mode=1(呼吸), color=4(红), brightness=150

// 绿色常亮
id(zw101_reader).set_rgb_led(3, 2, 200);  // mode=3(常亮), color=2(绿), brightness=200

// 关闭LED
id(zw101_reader).set_rgb_led(4, 0, 0);    // mode=4(关闭)
```

**重要提示**:
- ⚠️ `loop_times` 设置为 `0xFF` 确保LED效果持续运行
- ⚠️ 如果 LED 只维持几秒就恢复默认，检查 `loop_times` 是否为 `0`
- ✅ 当前代码已默认使用 `0xFF`，确保LED持续有效

### 确认码

| 确认码 | 含义 |
|-------|------|
| 0x00 | 成功 |
| 0x01 | 接收包错误 |
| 0x02 | 传感器上没有手指 |
| 0x03 | 录入指纹图像失败 |
| 0x06 | 指纹太乱而生成特征失败 |
| 0x07 | 特征点太少而生成特征失败 |
| 0x08 | 指纹不匹配 |
| 0x09 | 没搜索到指纹 |
| 0x0A | 特征合并失败 |
| 0x0B | 访问指纹库地址越界 |
| 0x10 | 删除模板失败 |
| 0x11 | 清空指纹库失败 |

---

## 版本信息

- **文档版本**: 1.0.0
- **组件版本**: ZW101 ESPHome Component
- **最后更新**: 2025
- **适用硬件**: ZW101 指纹识别模组
- **适用平台**: ESP32 系列 (ESP32-C3, ESP32-S3 等)

---

## 相关文档

- [ZW101 功能特性说明](FEATURES_ENHANCED.md)
- [ESPHome 官方文档](https://esphome.io)
- [组件使用示例](../../configs/actuators/fingerprint-zw101-new.yaml)

---

**技术支持**: 如有问题请参考故障排查章节或查看日志输出
