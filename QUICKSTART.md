# ZW101 指纹识别模组 - 快速开始

## 5 分钟快速部署

### 步骤 1: 硬件连接

```
ZW101 模组          ESP32-C3
──────────────────────────────
VCC   ──────────> 3.3V电源 
GND   ──────────> GND
TX    ──────────> GPIO0 (RX)
RX    ──────────> GPIO1 (TX)
```


### 步骤 2: 创建配置文件

创建 `fingerprint-zw101-new.yaml`:

```yaml
substitutions:
  device_name: fingerprint-zw101
  friendly_name: "ZW101 Fingerprint"

esphome:
  name: ${device_name}

esp32:
  board: airm2m_core_esp32c3
  framework:
    type: arduino

# WiFi 配置
wifi:
  ssid: "你的WiFi名称"
  password: "你的WiFi密码"

# API
api:

# OTA
ota:

# 日志
logger:
  level: DEBUG

# 引入 External Component
external_components:
  - source:
      type: local
      path: ../../components

# UART 配置
uart:
  id: fingerprint_uart
  tx_pin: GPIO1
  rx_pin: GPIO0
  baud_rate: 57600

# ZW101 组件
zw101:
  id: zw101_reader
  uart_id: fingerprint_uart

# 二值传感器
binary_sensor:
  - platform: zw101
    zw101_id: zw101_reader
    name: "${friendly_name} Match"

# 传感器
sensor:
  - platform: zw101
    zw101_id: zw101_reader
    match_score:
      name: "${friendly_name} Match Score"
    match_id:
      name: "${friendly_name} Match ID"

# 文本传感器
text_sensor:
  - platform: zw101
    zw101_id: zw101_reader
    status:
      name: "${friendly_name} Status"

# 开关
switch:
  - platform: zw101
    zw101_id: zw101_reader
    enroll:
      name: "${friendly_name} Enroll"
    clear:
      name: "${friendly_name} Clear Library"
```

### 步骤 3: 编译上传

```bash
esphome run fingerprint-zw101-new.yaml
```

### 步骤 4: 测试

1. 打开 Home Assistant
2. 找到 "ZW101 Fingerprint Enroll" 开关
3. 打开开关开始注册指纹
4. 按提示按压 5 次指纹
5. 注册成功后,测试匹配功能

## 完成! 🎉

现在你的指纹识别系统已经可以工作了!

## 下一步

### 添加服务接口 (可选)

在配置中添加:

```yaml
api:
  services:
    # 删除指纹
    - service: delete_fingerprint
      variables:
        fingerprint_id: int
      then:
        - lambda: |-
            id(zw101_reader).delete_fingerprint(fingerprint_id);

    # LED 控制
    - service: set_led
      variables:
        mode: int        # 1=呼吸 2=闪烁 3=常亮 4=关闭
        color: int       # 1=蓝 2=绿 4=红 7=白
        brightness: int  # 0-255
      then:
        - lambda: |-
            id(zw101_reader).set_rgb_led(mode, color, brightness);

    # 检查在线
    - service: check_online
      then:
        - lambda: |-
            id(zw101_reader).handshake();

    # 读取数量
    - service: read_count
      then:
        - lambda: |-
            id(zw101_reader).read_valid_template_count();
```

### 添加自动化 (可选)

```yaml
automation:
  # 匹配成功开门
  - alias: "Fingerprint Unlock"
    trigger:
      - platform: state
        entity_id: binary_sensor.zw101_fingerprint_match
        to: "on"
    action:
      - logger.log: "指纹匹配成功!"
      # 在这里添加开门动作

  # LED 反馈
  - alias: "Success LED"
    trigger:
      - platform: state
        entity_id: binary_sensor.zw101_fingerprint_match
        to: "on"
    action:
      - lambda: |-
          id(zw101_reader).set_rgb_led(3, 2, 200);  // 绿色常亮
      - delay: 3s
      - lambda: |-
          id(zw101_reader).set_rgb_led(4, 0, 0);    // 关灯
```

## 常见问题

### Q: 设备一直重启?
**A**: 已修复! 本组件使用完全非阻塞设计,不会导致看门狗超时。

### Q: 指纹识别率低?
**A**:
1. 确保手指清洁干燥
2. 注册时多次按压不同角度
3. 调整 `search` 检测间隔

### Q: 如何删除单个指纹?
**A**: 使用服务调用:
```yaml
service: esphome.zw101_reader_delete_fingerprint
data:
  fingerprint_id: 5  # 删除 ID 为 5 的指纹
```

### Q: RGB 灯不亮?
**A**:
1. 检查模组型号是否支持 RGB
2. 确认 VCC 供电充足 (5V)

### Q: 如何查看已注册数量?
**A**: 使用服务:
```yaml
service: esphome.zw101_reader_read_count
```

## 完整文档

- **README.md** - 完整说明
- **EXAMPLES.md** - 详细示例
- **FEATURES_ENHANCED.md** - 功能说明
- **BUGFIX.md** - 问题修复
- **SUMMARY.md** - 项目总结

## 功能列表

### 基础功能 ✅
- ✅ 指纹注册 (5次采集)
- ✅ 指纹匹配 (自动搜索)
- ✅ 清空指纹库
- ✅ 读取模组信息

### 高级功能 ✅
- ✅ 删除指定指纹
- ✅ 读取指纹数量
- ✅ RGB LED 控制 (7种模式 × 7种颜色)
- ✅ 握手测试
- ✅ 进入休眠模式
- ✅ 自动注册模式
- ✅ 自动匹配模式

### 架构特性 ✅
- ✅ 完全非阻塞
- ✅ 不会重启
- ✅ WiFi 稳定
- ✅ 连续搜索判断
- ✅ 状态机管理

## 技术支持

遇到问题? 查看:
1. **BUGFIX.md** - 常见问题修复
2. **日志输出** - 启用 DEBUG 日志
3. **文档** - 8 份详细文档

## LED 快速参考

```yaml
# 待机 - 蓝色呼吸
set_rgb_led(1, 1, 50)

# 成功 - 绿色常亮
set_rgb_led(3, 2, 200)

# 失败 - 红色闪烁
set_rgb_led(2, 4, 255)

# 关灯
set_rgb_led(4, 0, 0)
```

---

**开始使用吧! 🚀**
