# ZW101 完整使用示例

## 基础配置

```yaml
# fingerprint-zw101-new.yaml
substitutions:
  device_name: fingerprint-zw101
  friendly_name: "ZW101 Fingerprint"
  fingerprint_rx_pin: GPIO0
  fingerprint_tx_pin: GPIO1

esphome:
  name: ${device_name}

esp32:
  board: airm2m_core_esp32c3
  framework:
    type: arduino

<<: !include ../../common.yaml

logger:
  level: DEBUG

external_components:
  - source:
      type: local
      path: ../../components

uart:
  id: fingerprint_uart
  tx_pin: ${fingerprint_tx_pin}
  rx_pin: ${fingerprint_rx_pin}
  baud_rate: 57600

# ZW101 组件
zw101:
  id: zw101_reader
  uart_id: fingerprint_uart

# 传感器
binary_sensor:
  - platform: zw101
    zw101_id: zw101_reader
    name: "${friendly_name} Match"
    id: fp_match

sensor:
  - platform: zw101
    zw101_id: zw101_reader
    match_score:
      name: "${friendly_name} Match Score"
      id: fp_score
    match_id:
      name: "${friendly_name} Match ID"
      id: fp_id

text_sensor:
  - platform: zw101
    zw101_id: zw101_reader
    status:
      name: "${friendly_name} Status"
      id: fp_status

# 开关
switch:
  - platform: zw101
    zw101_id: zw101_reader
    enroll:
      name: "${friendly_name} Enroll"
      id: fp_enroll
    clear:
      name: "${friendly_name} Clear Library"
      id: fp_clear
```

## 高级功能示例

### 示例1: 完整的指纹门禁系统

```yaml
# 添加到上面的配置

# 服务定义 - 暴露所有功能到 Home Assistant
api:
  services:
    # 删除指定指纹
    - service: delete_fingerprint
      variables:
        fingerprint_id: int
      then:
        - lambda: |-
            ESP_LOGI("main", "Deleting fingerprint ID: %d", fingerprint_id);
            id(zw101_reader).delete_fingerprint(fingerprint_id);

    # RGB LED 控制
    - service: set_led
      variables:
        mode: int       # 1=呼吸 2=闪烁 3=常亮 4=关闭
        color: int      # 1=蓝 2=绿 4=红 7=白
        brightness: int # 0-255
      then:
        - lambda: |-
            id(zw101_reader).set_rgb_led(mode, color, brightness);

    # 进入休眠模式
    - service: enter_sleep
      then:
        - lambda: |-
            id(zw101_reader).enter_sleep_mode();

    # 自动注册模式
    - service: auto_enroll
      variables:
        timeout: int  # 超时时间(秒)
      then:
        - lambda: |-
            id(zw101_reader).auto_enroll_mode(timeout);

    # 自动匹配模式
    - service: auto_match
      then:
        - lambda: |-
            id(zw101_reader).auto_match_mode();

    # 取消自动模式
    - service: cancel_auto
      then:
        - lambda: |-
            id(zw101_reader).cancel_auto_mode();

    # 握手测试
    - service: check_online
      then:
        - lambda: |-
            if (id(zw101_reader).handshake()) {
              ESP_LOGI("main", "Module is online");
            } else {
              ESP_LOGE("main", "Module is offline!");
            }

    # 读取指纹数量
    - service: read_count
      then:
        - lambda: |-
            id(zw101_reader).read_valid_template_count();

# 自动化示例
automation:
  # ==================== 启动检查 ====================
  - alias: "Startup Check"
    trigger:
      - platform: homeassistant
        event: start
    action:
      - delay: 2s
      - lambda: |-
          ESP_LOGI("main", "Checking fingerprint module...");
          if (id(zw101_reader).handshake()) {
            ESP_LOGI("main", "Module online, reading info");
            id(zw101_reader).read_valid_template_count();
            // 设置待机蓝色灯光
            id(zw101_reader).set_rgb_led(1, 1, 50);
          } else {
            ESP_LOGE("main", "Module offline!");
          }

  # ==================== LED 视觉反馈 ====================
  # 注册开始 - 蓝色呼吸灯
  - alias: "Enroll Start - Blue Breath"
    trigger:
      - platform: state
        entity_id: switch.zw101_fingerprint_enroll
        to: "on"
    action:
      - lambda: |-
          id(zw101_reader).set_rgb_led(1, 1, 100);

  # 匹配成功 - 绿色常亮3秒
  - alias: "Match Success - Green Light"
    trigger:
      - platform: state
        entity_id: binary_sensor.zw101_fingerprint_match
        to: "on"
    action:
      - lambda: |-
          ESP_LOGI("main", "Fingerprint matched! ID: %d",
                   (int)id(fp_id).state);
          id(zw101_reader).set_rgb_led(3, 2, 200);  // 绿色常亮
      - delay: 3s
      - lambda: |-
          id(zw101_reader).set_rgb_led(4, 0, 0);    // 关灯

  # 验证失败 - 红色闪烁3次
  - alias: "Match Failed - Red Blink"
    trigger:
      - platform: state
        entity_id: sensor.zw101_fingerprint_status
        to: "No Valid Fingerprint"
    action:
      - lambda: |-
          for (int i = 0; i < 3; i++) {
            id(zw101_reader).set_rgb_led(3, 4, 255);  // 红色亮
            delay(300);
            id(zw101_reader).set_rgb_led(4, 0, 0);    // 关灯
            delay(300);
          }

  # ==================== 智能节能 ====================
  # 夜间进入休眠
  - alias: "Night Sleep Mode"
    trigger:
      - platform: time
        at: "23:00:00"
    action:
      - lambda: |-
          ESP_LOGI("main", "Entering sleep mode for night");
          id(zw101_reader).set_rgb_led(4, 0, 0);  // 关灯
          delay(500);
          id(zw101_reader).enter_sleep_mode();

  # 早晨唤醒
  - alias: "Morning Wakeup"
    trigger:
      - platform: time
        at: "07:00:00"
    action:
      - lambda: |-
          ESP_LOGI("main", "Waking up from sleep");
          id(zw101_reader).handshake();  // 唤醒模组
          id(zw101_reader).set_rgb_led(1, 1, 50);  // 待机蓝光

  # ==================== 安全功能 ====================
  # 多次失败后锁定(配合 Home Assistant)
  - alias: "Multiple Failures - Lockout"
    trigger:
      - platform: state
        entity_id: sensor.zw101_fingerprint_status
        to: "No Valid Fingerprint"
    condition:
      # 检查10分钟内失败次数
      - condition: template
        value_template: "{{ states('counter.fp_fail_count')|int > 5 }}"
    action:
      - lambda: |-
          ESP_LOGW("main", "Too many failures, entering lockout");
          id(zw101_reader).set_rgb_led(2, 4, 255);  // 红色快闪
          id(zw101_reader).enter_sleep_mode();      // 进入休眠
      - service: notify.all
        data:
          message: "指纹识别锁定 - 失败次数过多"

# ==================== 门锁控制 ====================
switch:
  - platform: gpio
    pin: GPIO4
    name: "${friendly_name} Door Lock"
    id: door_lock
    icon: "mdi:door"
    restore_mode: ALWAYS_OFF
    on_turn_on:
      - lambda: |-
          ESP_LOGI("main", "Door unlocked");
      - delay: 5s  # 5秒后自动关闭
      - switch.turn_off: door_lock
      - lambda: |-
          ESP_LOGI("main", "Door locked");

automation:
  # 指纹验证开门
  - alias: "Fingerprint Unlock Door"
    trigger:
      - platform: state
        entity_id: binary_sensor.zw101_fingerprint_match
        to: "on"
    action:
      - switch.turn_on: door_lock
      - service: notify.mobile_app
        data:
          message: "门已解锁 - 用户ID: {{ states('sensor.zw101_fingerprint_match_id') }}"
```

### 示例2: 多用户管理系统

```yaml
# Home Assistant 配置 (configuration.yaml)
input_select:
  fingerprint_users:
    name: 指纹用户
    options:
      - "0: 管理员"
      - "1: 爸爸"
      - "2: 妈妈"
      - "3: 孩子"
      - "4: 保姆"
    initial: "0: 管理员"

counter:
  fp_fail_count:
    name: 指纹失败次数
    icon: mdi:alert
    initial: 0
    step: 1

# 自动化
automation:
  # 匹配成功 - 记录用户
  - alias: "Log Fingerprint Access"
    trigger:
      - platform: state
        entity_id: binary_sensor.zw101_fingerprint_match
        to: "on"
    action:
      - service: logbook.log
        data:
          name: "指纹门禁"
          message: >
            用户 {{ states('sensor.zw101_fingerprint_match_id') }}
            验证成功 (得分: {{ states('sensor.zw101_fingerprint_match_score') }})
      - service: counter.reset
        target:
          entity_id: counter.fp_fail_count

  # 匹配失败 - 计数
  - alias: "Count Fingerprint Failures"
    trigger:
      - platform: state
        entity_id: sensor.zw101_fingerprint_status
        to: "No Valid Fingerprint"
    action:
      - service: counter.increment
        target:
          entity_id: counter.fp_fail_count

  # 删除选定用户
  - alias: "Delete Selected User"
    trigger:
      - platform: state
        entity_id: input_boolean.delete_fingerprint
        to: "on"
    action:
      - service: esphome.zw101_reader_delete_fingerprint
        data:
          fingerprint_id: >
            {{ states('input_select.fingerprint_users').split(':')[0]|int }}
      - service: input_boolean.turn_off
        target:
          entity_id: input_boolean.delete_fingerprint
```

### 示例3: 自动注册助手

```yaml
# ESPHome 配置
script:
  # 自动注册流程
  - id: auto_enroll_script
    mode: single
    then:
      - lambda: |-
          ESP_LOGI("main", "Starting auto enroll");
          id(zw101_reader).set_rgb_led(1, 1, 100);  // 蓝色呼吸
      - delay: 1s
      - lambda: |-
          id(zw101_reader).auto_enroll_mode(60);    // 60秒超时
      # 等待模组返回结果...
      - wait_until:
          condition:
            lambda: 'return !id(zw101_reader).auto_mode_active_;'
          timeout: 65s
      - lambda: |-
          ESP_LOGI("main", "Auto enroll completed");
          id(zw101_reader).set_rgb_led(3, 2, 150);  // 成功绿灯
      - delay: 2s
      - lambda: |-
          id(zw101_reader).set_rgb_led(4, 0, 0);    // 关灯

# 触发自动注册
button:
  - platform: template
    name: "自动注册指纹"
    icon: "mdi:fingerprint-add"
    on_press:
      - script.execute: auto_enroll_script
```

### 示例4: 智能提示音

```yaml
# 需要额外的蜂鸣器或扬声器
output:
  - platform: ledc
    pin: GPIO5
    id: buzzer_output

rtttl:
  output: buzzer_output

automation:
  # 匹配成功 - 成功提示音
  - alias: "Success Beep"
    trigger:
      - platform: state
        entity_id: binary_sensor.zw101_fingerprint_match
        to: "on"
    action:
      - rtttl.play: "success:d=4,o=5,b=100:16e6,16e6"

  # 匹配失败 - 错误提示音
  - alias: "Failure Beep"
    trigger:
      - platform: state
        entity_id: sensor.zw101_fingerprint_status
        to: "No Valid Fingerprint"
    action:
      - rtttl.play: "error:d=4,o=5,b=100:16c,16c"

  # 注册成功 - 欢迎音
  - alias: "Enroll Success Melody"
    trigger:
      - platform: state
        entity_id: sensor.zw101_fingerprint_status
        to: "Enroll Success (ID: *"
    action:
      - rtttl.play: "welcome:d=4,o=5,b=125:16c6,16g,16e,16c"
```

## 服务调用示例

### Home Assistant 服务调用

```yaml
# 删除指纹 ID 5
service: esphome.zw101_reader_delete_fingerprint
data:
  fingerprint_id: 5

# 设置紫色呼吸灯
service: esphome.zw101_reader_set_led
data:
  mode: 1       # 呼吸
  color: 5      # 紫色
  brightness: 120

# 进入休眠
service: esphome.zw101_reader_enter_sleep

# 启动60秒自动注册
service: esphome.zw101_reader_auto_enroll
data:
  timeout: 60

# 检查模组在线状态
service: esphome.zw101_reader_check_online

# 读取指纹数量
service: esphome.zw101_reader_read_count
```

## LED 颜色和模式参考

### 模式 (mode)
- 1 = 呼吸灯 (Breath)
- 2 = 闪烁 (Blink)
- 3 = 常亮 (On)
- 4 = 关闭 (Off)
- 5 = 渐变开 (Fade In)
- 6 = 渐变关 (Fade Out)
- 7 = 跑马灯 (Marquee)

### 颜色 (color)
- 0 = 关闭
- 1 = 蓝色 (Blue)
- 2 = 绿色 (Green)
- 3 = 青色 (Cyan)
- 4 = 红色 (Red)
- 5 = 紫色 (Magenta)
- 6 = 黄色 (Yellow)
- 7 = 白色 (White)

### 建议场景配色
- **待机**: 蓝色呼吸 `(1, 1, 50)`
- **注册中**: 蓝色呼吸 `(1, 1, 100)`
- **匹配中**: 蓝色闪烁 `(2, 1, 100)`
- **成功**: 绿色常亮 `(3, 2, 200)`
- **失败**: 红色闪烁 `(2, 4, 255)`
- **警告**: 黄色闪烁 `(2, 6, 150)`
- **休眠**: 关闭 `(4, 0, 0)`

## 故障排除

### 1. 服务不可用
确保在 `api:` 部分添加了服务定义。

### 2. RGB 灯不亮
- 检查模组是否支持 RGB (部分 ZW101 型号无 RGB)
- 确认 VCC 供电充足 (5V)

### 3. 自动模式无响应
- 检查超时设置是否合理
- 查看日志确认命令发送成功

### 4. 休眠后无法唤醒
- 使用 `handshake()` 唤醒模组
- 重新上电

## 完整功能列表

| 功能 | 方法 | 说明 |
|------|------|------|
| 握手测试 | `handshake()` | 检测模组在线 |
| 读取数量 | `read_valid_template_count()` | 读取已注册数量 |
| 删除指纹 | `delete_fingerprint(id)` | 删除指定ID |
| RGB控制 | `set_rgb_led(mode, color, bright)` | LED灯效 |
| 进入休眠 | `enter_sleep_mode()` | 省电模式 |
| 自动注册 | `auto_enroll_mode(timeout)` | 简化注册 |
| 自动匹配 | `auto_match_mode()` | 简化验证 |
| 取消自动 | `cancel_auto_mode()` | 退出自动模式 |

所有功能都是非阻塞的,不会导致系统重启!
