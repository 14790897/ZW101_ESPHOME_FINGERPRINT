# ZW101 休眠模式详解

## 🔄 默认工作模式

### ⚡ 永不休眠

**默认情况下,ZW101模组会持续工作,永不休眠!**

```cpp
// zw101.cpp:69-73
case SEARCH_IDLE:
  // 每1秒启动一次新搜索
  if (now - search_last_action_ > 1000) {
    search_state_ = SEARCH_GET_IMAGE;
    ...
  }
```

### 📊 工作流程

```
启动 → 1秒后开始 → 每秒搜索指纹 → 永不停止
         └─────────── ∞ 循环 ────────┘
```

| 时间 | 状态 | 动作 |
|------|------|------|
| 0秒 | 启动 | 初始化模组 |
| 0.5秒 | 关闭LED | 关闭默认灯光 |
| 1秒 | 开始搜索 | 读取模组信息 |
| 2秒 | 搜索中 | GET_IMAGE → GEN_CHAR → SEARCH |
| 3秒 | 搜索中 | GET_IMAGE → GEN_CHAR → SEARCH |
| 4秒 | 搜索中 | GET_IMAGE → GEN_CHAR → SEARCH |
| ... | ... | **每秒重复** |
| ∞ | 永不休眠 | 持续搜索 |

## 💤 休眠模式

### 什么是休眠?

当调用 `enter_sleep_mode()` 后:
1. **ZW101模组进入低功耗状态**
2. **停止指纹搜索** (不再每秒扫描)
3. **LED关闭**
4. **功耗降低约80%** (典型值: 工作~100mA → 休眠~20mA)

```cpp
// zw101.cpp:56-61
if (auto_mode_active_ || sleep_mode_) {
  return;  // 🛑 不进行搜索
}

// 处理搜索流程
process_search();  // ⏭️ 只有非休眠时才执行
```

### 如何进入休眠?

#### 方法1: Home Assistant 服务调用
```yaml
service: esphome.fingerprint_zw101_enter_sleep
```

#### 方法2: ESPHome 自动化 (夜间休眠)
```yaml
time:
  - platform: homeassistant
    id: homeassistant_time

automation:
  - alias: "Night Sleep Mode"
    trigger:
      - platform: time
        at: "23:00:00"
    action:
      - lambda: |-
          id(zw101_reader).enter_sleep_mode();

  - alias: "Morning Wakeup"
    trigger:
      - platform: time
        at: "07:00:00"
    action:
      - lambda: |-
          id(zw101_reader).handshake();  // 唤醒
```

#### 方法3: 通过开关控制 (可选添加)
```yaml
switch:
  - platform: template
    name: "Fingerprint Sleep Mode"
    id: fp_sleep_switch
    optimistic: true
    turn_on_action:
      - lambda: |-
          id(zw101_reader).enter_sleep_mode();
    turn_off_action:
      - lambda: |-
          id(zw101_reader).handshake();  // 唤醒
```

### 如何唤醒?

#### 方法1: 握手命令
```cpp
id(zw101_reader).handshake();
```

#### 方法2: Home Assistant 服务
```yaml
service: esphome.fingerprint_zw101_check_online
```

#### 方法3: 任何UART命令
发送任何命令都会唤醒模组:
- `read_fp_info()`
- `read_valid_template_count()`
- `register_fingerprint()`
- 等等...

## 🎯 实用休眠方案

### 方案1: 夜间定时休眠 ⏰
**适用场景**: 门禁系统,夜间无人使用

```yaml
# 23:00-07:00 休眠
automation:
  - alias: "Night Sleep"
    trigger:
      - platform: time
        at: "23:00:00"
    action:
      - logger.log: "Good night! Entering sleep mode"
      - lambda: id(zw101_reader).enter_sleep_mode();

  - alias: "Morning Wakeup"
    trigger:
      - platform: time
        at: "07:00:00"
    action:
      - logger.log: "Good morning! Waking up"
      - lambda: id(zw101_reader).handshake();
```

**省电效果**:
- 工作16小时,休眠8小时
- 日均功耗降低约 **27%**

### 方案2: 人体感应休眠 👤
**适用场景**: 配合人体传感器,无人时休眠

```yaml
# 需要添加PIR人体传感器
binary_sensor:
  - platform: gpio
    pin: GPIO5
    name: "PIR Motion"
    id: pir_motion
    device_class: motion

automation:
  # 5分钟无人 → 休眠
  - alias: "Auto Sleep on No Motion"
    trigger:
      - platform: state
        entity_id: binary_sensor.pir_motion
        to: "off"
        for:
          minutes: 5
    action:
      - logger.log: "No motion for 5min, sleeping..."
      - lambda: id(zw101_reader).enter_sleep_mode();

  # 检测到人 → 唤醒
  - alias: "Auto Wakeup on Motion"
    trigger:
      - platform: state
        entity_id: binary_sensor.pir_motion
        to: "on"
    action:
      - logger.log: "Motion detected! Waking up"
      - lambda: id(zw101_reader).handshake();
```

**省电效果**:
- 根据人流量动态调整
- 人少场景可节省 **50-80%** 功耗

### 方案3: 空闲超时休眠 ⏱️
**适用场景**: 长时间无操作时自动休眠

```yaml
automation:
  # 10分钟无匹配 → 休眠
  - alias: "Auto Sleep on Idle"
    trigger:
      - platform: state
        entity_id: binary_sensor.zw101_fingerprint_match
        to: "off"
        for:
          minutes: 10
    action:
      - logger.log: "Idle for 10min, entering sleep"
      - lambda: id(zw101_reader).enter_sleep_mode();

  # 任何操作 → 唤醒
  - alias: "Auto Wakeup on Activity"
    trigger:
      - platform: state
        entity_id: text_sensor.zw101_fingerprint_status
    action:
      - lambda: id(zw101_reader).handshake();
```

**省电效果**:
- 灵活响应使用频率
- 适合办公场所,午休/下班后自动休眠

### 方案4: 手动控制休眠 🎚️
**适用场景**: 需要临时禁用指纹识别

```yaml
switch:
  - platform: template
    name: "Fingerprint Active"
    id: fp_active_switch
    optimistic: true
    restore_mode: RESTORE_DEFAULT_ON
    turn_on_action:
      - logger.log: "Fingerprint reader activated"
      - lambda: id(zw101_reader).handshake();
    turn_off_action:
      - logger.log: "Fingerprint reader deactivated"
      - lambda: id(zw101_reader).enter_sleep_mode();
```

**使用场景**:
- 临时禁用门禁
- 维护/清洁时关闭
- 配合其他自动化逻辑

## ⚠️ 注意事项

### 1. 休眠状态下不能注册/匹配

```
休眠中 → 按手指 → ❌ 无响应
       → 先唤醒 → ✅ 可以使用
```

### 2. 唤醒需要时间

```cpp
id(zw101_reader).handshake();  // 发送唤醒命令
delay(100);                     // 等待100ms
// 现在可以使用
```

建议在唤醒后等待 **100-200ms** 再进行操作。

### 3. 休眠状态会重置

以下情况会自动唤醒模组:
- ESP32 重启
- 发送任何UART命令
- 调用 `handshake()` / `read_fp_info()` 等

### 4. 功耗对比

| 状态 | 典型功耗 | 说明 |
|------|----------|------|
| 工作搜索中 | ~100mA | 每秒扫描指纹 |
| 休眠模式 | ~20mA | 低功耗待机 |
| 完全断电 | 0mA | 需要重新初始化 |

## 🔍 休眠状态检测

### 通过状态传感器
```yaml
text_sensor:
  - platform: zw101
    status:
      name: "Fingerprint Status"
      # 显示: "Sleep Mode" / "Ready" / "Searching" 等
```

### 通过日志
```
[I][zw101:498] Module entered sleep mode
[I][zw101:363] Handshake successful  # 表示已唤醒
```

## 📝 总结

| 项目 | 默认行为 | 推荐配置 |
|------|----------|----------|
| 启动状态 | 立即开始搜索 | ✅ 保持默认 |
| 待机行为 | 每秒搜索,永不休眠 | ⚠️ 可选休眠 |
| 夜间模式 | 持续工作 | ✅ 建议休眠 |
| 无人时 | 持续工作 | ✅ 建议休眠 |
| 唤醒方式 | - | `handshake()` |
| 省电效果 | 0% | 20-80% |

### 最佳实践

1. **家庭门禁**: 夜间定时休眠 (23:00-07:00)
2. **办公室**: 人体感应休眠 (无人5分钟后)
3. **仓库/车库**: 空闲超时休眠 (10分钟无操作)
4. **临时禁用**: 手动开关控制

### 配置示例

已在 `fingerprint-zw101-new.yaml` 文件末尾提供了完整的注释示例 (304-388行),根据需要取消注释即可使用!

---

**版本**: 1.0.0
**日期**: 2025-10-20
**相关文件**: `zw101.cpp`, `zw101.h`, `fingerprint-zw101-new.yaml`
