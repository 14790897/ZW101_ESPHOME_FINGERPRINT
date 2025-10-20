# ZW101 指纹识别模组 ESPHome External Component

## 概述

本项目是基于 ESPHome **External Component** 标准构建的 ZW101 光学指纹识别模组驱动,符合 ESPHome 2025.2.0+ 版本要求,替代了已废弃的 Custom Component 机制。

## 核心特性

### ✅ 符合 ESPHome 最新标准
- 使用 **External Component** 架构
- 完整的 YAML 配置验证
- 模块化的 Python 代码生成
- 标准的 C++ 组件实现

### ✅ 连续搜索判断算法
- 自动进行最多 **5次连续尝试** 来获取和识别指纹
- 每次失败后智能等待再重试
- 只有在成功生成有效特征后才进行库搜索
- 避免误触发,提高识别准确率

### ✅ 完整功能支持
- **自动搜索**: 每秒自动检测指纹并匹配
- **注册指纹**: 通过开关触发,非阻塞式采集5次指纹样本
- **清空指纹库**: 一键删除所有已注册指纹
- **状态反馈**: 实时显示匹配ID、得分、状态信息
- **模组信息**: 启动时自动读取已注册指纹数量

## 目录结构

```
MY-ESPHOME/
├── components/                    # External Components 目录
│   └── zw101/                    # ZW101 组件
│       ├── __init__.py           # 主配置验证和代码生成
│       ├── binary_sensor.py      # Binary Sensor 平台
│       ├── sensor.py             # Sensor 平台
│       ├── text_sensor.py        # Text Sensor 平台
│       ├── switch.py             # Switch 平台
│       ├── zw101.h               # C++ 头文件
│       └── zw101.cpp             # C++ 实现文件
│
└── configs/actuators/
    ├── fingerprint-zw101-new.yaml  # 新版配置文件
    └── README_ZW101.md             # 本文档
```

## 硬件连接

```
ZW101 模组          ESP32-C3
----------------------------------
VCC   ────────────> 独立5V电源 (重要!)
GND   ────────────> GND
TX    ────────────> GPIO0 (RX)
RX    ────────────> GPIO1 (TX)
```

⚠️ **重要提示**: ZW101 的 VCC 必须使用独立 5V 电源供电,不能仅靠 ESP32 供电!

## 配置说明

### 1. 引入 External Component

```yaml
external_components:
  - source:
      type: local
      path: ../../components  # 相对于 YAML 文件的路径
```

### 2. 配置 UART

```yaml
uart:
  id: fingerprint_uart
  tx_pin: GPIO1
  rx_pin: GPIO0
  baud_rate: 57600  # ZW101 默认波特率
```

### 3. 配置 ZW101 组件

```yaml
zw101:
  id: zw101_reader
  uart_id: fingerprint_uart
```

### 4. 配置传感器和开关

```yaml
# 指纹匹配状态
binary_sensor:
  - platform: zw101
    zw101_id: zw101_reader
    name: "Fingerprint Match"

# 匹配得分和ID
sensor:
  - platform: zw101
    zw101_id: zw101_reader
    match_score:
      name: "Match Score"
    match_id:
      name: "Match ID"

# 状态文本
text_sensor:
  - platform: zw101
    zw101_id: zw101_reader
    status:
      name: "Status"

# 控制开关
switch:
  - platform: zw101
    zw101_id: zw101_reader
    enroll:
      name: "Enroll Fingerprint"
    clear:
      name: "Clear Library"
```

## 使用方法

### 1. 编译和上传

```bash
esphome run configs/actuators/fingerprint-zw101-new.yaml
```

### 2. 注册指纹

在 Home Assistant 中:
1. 打开开关 **"ZW101 Fingerprint Enroll"**
2. 观察状态传感器,按提示依次按压指纹 **5次**
3. 每次按压后需移开手指,等待提示再次按压
4. 状态显示 "Enroll Success (ID: X)" 表示成功

日志输出示例:
```
[I][zw101:xxx] Starting fingerprint enrollment
[I][zw101:xxx] Place finger (sample 1/5)
[I][zw101:xxx] Finger detected, capturing sample 1/5
[I][zw101:xxx] Sample 1 captured
[I][zw101:xxx] Remove finger and place again (1/5)
...
[I][zw101:xxx] Fingerprint enrolled successfully as ID 1
```

### 3. 验证指纹

指纹会 **自动每秒检测一次**:
- 将手指放在模组上
- 系统自动进行最多5次连续尝试识别
- 匹配成功时:
  - `binary_sensor` 变为 **ON**
  - `match_id` 显示匹配的指纹ID
  - `match_score` 显示匹配得分
  - 日志输出: `[I][zw101:xxx] Match found! ID: 1, Score: 150`

### 4. 清空指纹库

⚠️ 此操作将删除所有已注册指纹!

在 Home Assistant 中打开 **"ZW101 Fingerprint Clear Library"** 开关

或通过服务调用:
```yaml
service: switch.turn_on
target:
  entity_id: switch.zw101_fingerprint_clear_library
```

## 核心代码解析

### 连续搜索逻辑 (zw101.cpp:145-195)

```cpp
bool ZW101Component::search_fingerprint() {
  int search_cnt = 0;
  uint8_t buffer_id = 1;

  // 最多尝试5次连续搜索
  while (search_cnt <= 5) {
    // 步骤1: 获取图像
    send_cmd(CMD_GET_IMAGE);
    if (receive_response()) {
      // 图像获取成功
    } else {
      delay(1000);  // 没有指纹,等待后继续
      continue;
    }

    // 步骤2: 生成特征
    send_cmd2(CMD_GEN_CHAR, buffer_id);
    if (receive_response()) {
      break;  // 成功生成特征,跳出循环
    } else {
      search_cnt++;  // 失败计数
      delay(500);    // 等待后重试
      continue;
    }
  }

  if (search_cnt > 5) return false;

  // 步骤3: 搜索指纹库
  send_search_cmd(buffer_id, 1, 1);
  // ... 处理响应
}
```

### 非阻塞注册流程 (zw101.cpp:46-141)

使用状态机实现非阻塞式注册,避免长时间阻塞主循环:

```cpp
enum EnrollState {
  ENROLL_IDLE,         // 空闲
  ENROLL_WAIT_FINGER,  // 等待手指
  ENROLL_CAPTURING,    // 捕获特征
  ENROLL_WAIT_REMOVE,  // 等待移开
  ENROLL_MERGING,      // 合并特征
  ENROLL_STORING       // 存储模板
};
```

## 自动化示例

### 指纹开锁

```yaml
automation:
  - alias: "指纹验证开锁"
    trigger:
      - platform: state
        entity_id: binary_sensor.zw101_fingerprint_match
        to: "on"
    condition:
      - condition: template
        value_template: "{{ trigger.to_state.attributes.match_id is defined }}"
    action:
      - service: switch.turn_on
        target:
          entity_id: switch.door_lock
      - service: notify.mobile_app
        data:
          message: "指纹验证成功 (ID: {{ state_attr('binary_sensor.zw101_fingerprint_match', 'match_id') }})"
```

### 指纹ID识别不同用户

```yaml
automation:
  - alias: "家庭成员识别"
    trigger:
      - platform: state
        entity_id: sensor.zw101_fingerprint_match_id
    action:
      - choose:
          - conditions:
              - condition: template
                value_template: "{{ trigger.to_state.state == '1' }}"
            sequence:
              - service: notify.all
                data:
                  message: "爸爸回家了"
          - conditions:
              - condition: template
                value_template: "{{ trigger.to_state.state == '2' }}"
            sequence:
              - service: notify.all
                data:
                  message: "妈妈回家了"
```

## External Component 架构优势

### vs. 旧版 Custom Component

| 特性 | Custom Component (已废弃) | External Component |
|------|---------------------------|-------------------|
| YAML 验证 | ❌ 无 | ✅ 完整验证 |
| 类型检查 | ❌ 弱 | ✅ 强类型 |
| 配置灵活性 | ❌ 需修改 C++ | ✅ 纯 YAML 配置 |
| 错误提示 | ❌ 编译时才发现 | ✅ 配置时即检查 |
| 维护性 | ❌ 手动复制文件 | ✅ 版本控制 |

### 开发结构

```
Python 层 (配置 & 代码生成)
  ├── __init__.py         - 主组件定义
  ├── binary_sensor.py    - Binary Sensor 平台
  ├── sensor.py           - Sensor 平台
  ├── text_sensor.py      - Text Sensor 平台
  └── switch.py           - Switch 平台

C++ 层 (运行时逻辑)
  ├── zw101.h             - 类定义和接口
  └── zw101.cpp           - 实现代码
```

## 协议参考

ZW101 使用串口通信,数据包格式:

```
[0xEF 0x01] [地址4字节] [包标识] [长度2字节] [指令码] [参数...] [校验和2字节]
```

主要指令码:
- `0x01` - 获取指纹图像 (CMD_GET_IMAGE)
- `0x02` - 生成特征文件 (CMD_GEN_CHAR)
- `0x04` - 搜索指纹库 (CMD_SEARCH)
- `0x05` - 合并特征文件 (CMD_REG_MODEL)
- `0x06` - 存储模板 (CMD_STORE_CHAR)
- `0x0D` - 清空指纹库 (CMD_CLEAR_LIB)
- `0x0F` - 读取系统参数 (CMD_READ_SYSPARA)

## 故障排除

### 编译错误

**错误**: `Component 'zw101' not found`
- 检查 `external_components` 路径配置是否正确
- 确认 `components/zw101/` 目录存在
- 确认 `__init__.py` 文件存在

**错误**: `No module named 'esphome.components.zw101'`
- 清理 ESPHome 缓存: `esphome clean <config>`
- 重新编译

### 运行时问题

**问题**: 模组无响应
- 检查 VCC 是否使用独立5V电源
- 检查串口引脚连接 (TX↔RX 交叉连接)
- 检查波特率设置 (默认57600)
- 查看日志是否有 UART 错误

**问题**: 识别率低
- 确保手指清洁干燥
- 注册时采集不同角度的指纹
- 检查光学窗口是否清洁

**问题**: 注册超时
- 默认超时30秒,可在 `zw101.cpp:76` 修改
- 确保及时按压手指

## 开发者信息

### 基于原始代码

- 原始 Arduino 代码: `configs/actuators/ZW101.ino`
- 改进为 ESPHome External Component
- 符合 ESPHome 2025.2.0+ 标准

### 主要改进

- ✅ 使用 External Component 架构
- ✅ 完整的 YAML 配置验证
- ✅ 非阻塞式注册流程(状态机)
- ✅ 保留连续搜索判断逻辑
- ✅ 模块化的传感器和开关
- ✅ 完善的日志输出

### 参考文档

- [ESPHome External Components](https://esphome.io/components/external_components/)
- [About removal of Custom Components](https://developers.esphome.io/blog/2025/02/19/about-the-removal-of-support-for-custom-components/)
- [ESPHome UART Component](https://esphome.io/components/uart.html)

## 贡献

欢迎提交 Issue 和 Pull Request!

## 许可

MIT License
