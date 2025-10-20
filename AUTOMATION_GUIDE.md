# Automation 配置方式说明

## 📝 两种配置方式

### 方式1: ESPHome YAML 配置 ✅
**位置**: `fingerprint-zw101-new.yaml` 文件中

**特点**:
- ✅ 配置在一个文件中
- ✅ 独立运行,不依赖Home Assistant连接
- ✅ 离线也能工作
- ⚠️ 定时功能需要用 `interval` + `wait_until`

**示例**:
```yaml
# 在 ESPHome YAML 文件中直接添加

time:
  - platform: homeassistant
    id: homeassistant_time

interval:
  # 每天23:00进入休眠
  - interval: 24h
    then:
      - wait_until:
          condition:
            lambda: |-
              auto time_now = id(homeassistant_time).now();
              return time_now.hour == 23 && time_now.minute == 0;
      - logger.log: "Entering sleep mode"
      - lambda: |-
          id(zw101_reader).enter_sleep_mode();
```

### 方式2: Home Assistant 配置 ✅
**位置**: Home Assistant 的 `automations.yaml` 或 UI界面

**特点**:
- ✅ 语法简洁,易于理解
- ✅ 可视化界面编辑
- ✅ 易于管理和调试
- ⚠️ 需要ESPHome与HA保持连接

**示例**:
```yaml
# 在 Home Assistant 的 automations.yaml 中添加

automation:
  - alias: "Fingerprint Night Sleep"
    trigger:
      - platform: time
        at: "23:00:00"
    action:
      - service: esphome.fingerprint_zw101_enter_sleep

  - alias: "Fingerprint Morning Wakeup"
    trigger:
      - platform: time
        at: "07:00:00"
    action:
      - service: esphome.fingerprint_zw101_check_online
```

## 🔍 详细对比

| 特性 | ESPHome 配置 | Home Assistant 配置 |
|------|--------------|---------------------|
| **配置位置** | `fingerprint-zw101-new.yaml` | `automations.yaml` 或 UI |
| **语法** | ESPHome YAML | HA Automation |
| **定时触发** | `interval` + `wait_until` | `platform: time` |
| **状态触发** | `on_state` | `platform: state` |
| **离线运行** | ✅ 支持 | ❌ 需要连接 |
| **可视化编辑** | ❌ 不支持 | ✅ 支持 |
| **调试** | 查看ESPHome日志 | HA界面追踪 |
| **复杂度** | 中等 | 简单 |

## 📋 使用场景推荐

### ESPHome 配置 (方式1)

✅ **适合场景**:
- 简单的定时任务 (夜间休眠)
- 希望离线也能工作
- 不想依赖Home Assistant
- 配置集中管理

❌ **不适合场景**:
- 需要复杂的条件判断
- 需要配合多个HA设备
- 需要频繁修改调试

**推荐用于**:
- ✅ 夜间定时休眠/唤醒
- ✅ LED灯光控制
- ✅ 简单的状态响应

### Home Assistant 配置 (方式2)

✅ **适合场景**:
- 复杂的自动化逻辑
- 需要配合其他HA设备 (如人体传感器)
- 需要可视化界面编辑
- 需要灵活调试

❌ **不适合场景**:
- 希望离线运行
- 不想依赖HA连接

**推荐用于**:
- ✅ 人体感应休眠 (配合PIR传感器)
- ✅ 多设备联动 (配合门锁、灯光等)
- ✅ 复杂条件判断
- ✅ 通知推送

## 🎯 实际配置示例

### 示例1: 夜间定时休眠

#### ESPHome 方式
```yaml
# fingerprint-zw101-new.yaml

time:
  - platform: homeassistant
    id: homeassistant_time

interval:
  - interval: 24h
    then:
      - wait_until:
          condition:
            lambda: |-
              auto time_now = id(homeassistant_time).now();
              return time_now.hour == 23 && time_now.minute == 0;
      - lambda: id(zw101_reader).enter_sleep_mode();

  - interval: 24h
    then:
      - wait_until:
          condition:
            lambda: |-
              auto time_now = id(homeassistant_time).now();
              return time_now.hour == 7 && time_now.minute == 0;
      - lambda: id(zw101_reader).handshake();
```

#### Home Assistant 方式
```yaml
# automations.yaml

automation:
  - alias: "Fingerprint Night Sleep"
    trigger:
      - platform: time
        at: "23:00:00"
    action:
      - service: esphome.fingerprint_zw101_enter_sleep

  - alias: "Fingerprint Morning Wakeup"
    trigger:
      - platform: time
        at: "07:00:00"
    action:
      - service: esphome.fingerprint_zw101_check_online
```

### 示例2: 人体感应休眠

#### ESPHome 方式 (不推荐,复杂)
```yaml
# 需要先添加PIR传感器到ESPHome

binary_sensor:
  - platform: gpio
    pin: GPIO5
    name: "PIR Motion"
    id: pir_motion
    on_state:
      then:
        - if:
            condition:
              binary_sensor.is_off: pir_motion
            then:
              - delay: 5min
              - lambda: id(zw101_reader).enter_sleep_mode();
        - if:
            condition:
              binary_sensor.is_on: pir_motion
            then:
              - lambda: id(zw101_reader).handshake();
```

#### Home Assistant 方式 ✅ **推荐**
```yaml
# automations.yaml

automation:
  - alias: "Fingerprint Auto Sleep on No Motion"
    trigger:
      - platform: state
        entity_id: binary_sensor.pir_motion
        to: "off"
        for:
          minutes: 5
    action:
      - service: esphome.fingerprint_zw101_enter_sleep

  - alias: "Fingerprint Auto Wakeup on Motion"
    trigger:
      - platform: state
        entity_id: binary_sensor.pir_motion
        to: "on"
    action:
      - service: esphome.fingerprint_zw101_check_online
```

## 📚 如何配置

### 在 ESPHome 中配置

1. **编辑文件**: `fingerprint-zw101-new.yaml`
2. **取消注释**: 找到第309-342行的注释部分
3. **移除 `#`**: 删除行首的 `#` 符号
4. **编译上传**: `esphome run fingerprint-zw101-new.yaml`

### 在 Home Assistant 中配置

#### 方法1: UI 界面 (推荐)
1. 打开 Home Assistant
2. 进入 **设置** → **自动化与场景** → **自动化**
3. 点击 **创建自动化**
4. 选择 **从空白开始**
5. 配置触发器和动作:
   - **触发器**: 时间 → 23:00:00
   - **动作**: 调用服务 → `esphome.fingerprint_zw101_enter_sleep`
6. 保存

#### 方法2: YAML 文件
1. 编辑 `/config/automations.yaml`
2. 添加自动化配置 (参考上面示例)
3. 重新加载自动化:
   - **开发者工具** → **YAML** → **自动化**

## ⚠️ 常见错误

### 错误1: 在 ESPHome 中使用 HA automation 语法
```yaml
# ❌ 错误 - 这是HA语法,不能放在ESPHome中
automation:
  - alias: "xxx"
    trigger:
      - platform: time
        at: "23:00:00"
```

应该使用 ESPHome 的 `interval` 或 `on_...` 触发器。

### 错误2: 在 HA 中使用 ESPHome lambda
```yaml
# ❌ 错误 - HA不支持lambda
action:
  - lambda: |-
      id(zw101_reader).enter_sleep_mode();
```

应该使用服务调用:
```yaml
action:
  - service: esphome.fingerprint_zw101_enter_sleep
```

### 错误3: 服务名称错误
```yaml
# ❌ 错误
service: zw101.enter_sleep

# ✅ 正确
service: esphome.fingerprint_zw101_enter_sleep
#        ^^^^^^^ ESPHome服务前缀
#                ^^^^^^^^^^^^^^^^^ 设备名_服务名
```

## 🎓 推荐学习路径

### 新手推荐: Home Assistant 配置
1. 界面友好,容易理解
2. 可视化编辑,即时生效
3. 错误提示明确

### 进阶推荐: ESPHome 配置
1. 配置集中,易于管理
2. 离线运行,更稳定
3. 学习ESPHome自动化机制

## 📝 总结

| 需求 | 推荐方式 | 理由 |
|------|----------|------|
| 夜间定时休眠 | Home Assistant | 语法简洁 |
| 人体感应休眠 | Home Assistant | 易于配合传感器 |
| 离线运行 | ESPHome | 不依赖连接 |
| 快速测试 | Home Assistant | 可视化调试 |
| 配置集中管理 | ESPHome | 一个文件搞定 |

**大多数情况下,推荐使用 Home Assistant 配置 (方式2)!**

---

**相关文件**:
- ESPHome配置: `fingerprint-zw101-new.yaml` (309-409行)
- HA配置示例: 见上方"实际配置示例"
- 完整文档: `SLEEP_MODE_EXPLAINED.md`
