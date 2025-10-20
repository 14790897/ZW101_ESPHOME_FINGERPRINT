# ZW101 增强功能说明

## 新增功能 (基于 fp_syno_protocol.c)

### 1. 读取有效模板个数 ✅
```cpp
bool read_valid_template_count();
```

**功能**: 快速读取已注册指纹数量
**命令码**: 0x1D
**返回**: 成功返回 true,并更新 status_sensor

**使用场景**:
- 快速检查已注册指纹数量
- 比 read_fp_info() 更快

**YAML 使用**:
```yaml
# 在自动化中调用
automation:
  - alias: "Check fingerprint count"
    trigger:
      - platform: homeassistant
        event: start
    action:
      - service: esphome.zw101_reader_read_count
```

### 2. 握手测试 ✅
```cpp
bool handshake();
```

**功能**: 测试模组是否在线
**命令码**: 0x35
**返回**: 成功返回 true,模组在线

**使用场景**:
- 检测模组连接状态
- 启动时验证通信
- 故障诊断

**特点**:
- 超时时间短 (500ms)
- 快速响应
- 更新状态传感器

### 3. 删除指定指纹 ✅
```cpp
bool delete_fingerprint(uint16_t id);
```

**功能**: 删除单个指纹
**命令码**: 0x0C
**参数**:
- `id` - 要删除的指纹ID (0-199)

**返回**: 成功返回 true

**使用场景**:
- 删除单个用户指纹
- 不影响其他指纹
- 比清空库更精确

**YAML 使用**:
```yaml
# Lambda 调用
lambda: |-
  id(zw101_reader).delete_fingerprint(5);  // 删除ID为5的指纹
```

### 4. RGB LED 控制 ✅
```cpp
void set_rgb_led(uint8_t mode, uint8_t color, uint8_t brightness = 100);
```

**功能**: 控制模组 RGB 灯效
**命令码**: 0x3C

**参数**:
- `mode` - 灯效模式:
  - 1 = 呼吸灯
  - 2 = 闪烁
  - 3 = 常亮
  - 4 = 关闭
  - 5 = 渐变开
  - 6 = 渐变关
  - 7 = 跑马灯

- `color` - 颜色:
  - 1 = 蓝色
  - 2 = 绿色
  - 3 = 青色
  - 4 = 红色
  - 5 = 紫色
  - 6 = 黄色
  - 7 = 白色

- `brightness` - 亮度 (0-255,默认100)

**使用场景**:
- 指纹录入时显示蓝色呼吸灯
- 验证成功显示绿色常亮
- 验证失败显示红色闪烁
- 待机时关闭LED节能

**YAML 使用**:
```yaml
automation:
  - alias: "指纹匹配成功 - 绿灯"
    trigger:
      - platform: state
        entity_id: binary_sensor.zw101_fingerprint_match
        to: "on"
    action:
      - lambda: |-
          id(zw101_reader).set_rgb_led(3, 2, 150);  // 绿色常亮

  - alias: "开始注册 - 蓝色呼吸"
    trigger:
      - platform: state
        entity_id: switch.zw101_fingerprint_enroll
        to: "on"
    action:
      - lambda: |-
          id(zw101_reader).set_rgb_led(1, 1, 100);  // 蓝色呼吸
```

## 完整命令码对比

| 命令码 | 功能 | ZW101.ino | fp_syno | ESPHome组件 |
|--------|------|-----------|---------|-------------|
| 0x01 | 获取图像 | ✅ | ✅ | ✅ |
| 0x02 | 生成特征 | ✅ | ✅ | ✅ |
| 0x03 | 精确比对 | ✅ | ❌ | ✅ |
| 0x04 | 搜索指纹 | ✅ | ✅ | ✅ |
| 0x05 | 合并特征 | ✅ | ✅ | ✅ |
| 0x06 | 存储模板 | ✅ | ✅ | ✅ |
| 0x0C | 删除模板 | ❌ | ✅ | ✅ **新增** |
| 0x0D | 清空指纹库 | ✅ | ✅ | ✅ |
| 0x0E | 写系统参数 | ❌ | ❌ | ⏳ 待实现 |
| 0x0F | 读系统参数 | ✅ | ❌ | ✅ |
| 0x1D | 读有效数量 | ❌ | ❌ | ✅ **新增** |
| 0x1F | 读索引表 | ❌ | ✅ | ⏳ 待实现 |
| 0x31 | 自动注册 | ❌ | ❌ | ⏳ 待实现 |
| 0x32 | 自动匹配 | ❌ | ❌ | ⏳ 待实现 |
| 0x33 | 进入休眠 | ❌ | ✅ | ⏳ 待实现 |
| 0x35 | 握手 | ❌ | ❌ | ✅ **新增** |
| 0x3C | RGB控制 | ❌ | ✅ | ✅ **新增** |

## 协议增强

### 1. 完整的错误码定义
从 fp_syno_protocol.h 移植了完整的错误码:
```cpp
#define PS_OK                0x00  // 成功
#define PS_COMM_ERR          0x01  // 通信错误
#define PS_NO_FINGER         0x02  // 无指纹
#define PS_GET_IMG_ERR       0x03  // 图像采集失败
#define PS_FP_TOO_DRY        0x04  // 指纹太干
#define PS_FP_TOO_WET        0x05  // 指纹太湿
// ... 更多错误码
```

### 2. RGB 灯效参数
```cpp
// 灯效模式
enum RGBMode {
  RGB_BREATH = 1,      // 呼吸灯
  RGB_BLINK = 2,       // 闪烁
  RGB_ON = 3,          // 常亮
  RGB_OFF = 4,         // 关闭
  RGB_FADE_IN = 5,     // 渐变开
  RGB_FADE_OUT = 6,    // 渐变关
  RGB_MARQUEE = 7      // 跑马灯
};

// 颜色定义
enum RGBColor {
  COLOR_BLUE = 1,      // 蓝色
  COLOR_GREEN = 2,     // 绿色
  COLOR_CYAN = 3,      // 青色
  COLOR_RED = 4,       // 红色
  COLOR_MAGENTA = 5,   // 紫色
  COLOR_YELLOW = 6,    // 黄色
  COLOR_WHITE = 7      // 白色
};
```
