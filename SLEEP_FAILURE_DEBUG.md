# ZW101 休眠命令失败问题排查指南

## 问题现象

```
[13:15:09.662][I][main:221]: Entering sleep mode
[13:15:10.048][W][zw101:505]: Failed to enter sleep mode
```

休眠命令在 400ms 后失败。

---

## 可能的原因

### 原因1: 模组正在执行其他操作

**症状**: 模组忙于搜索指纹、处理其他命令等

**解决方案**: 在进入休眠前停止所有操作

```yaml
button:
  - platform: template
    name: "Enter Sleep (Safe)"
    on_press:
      - lambda: |-
          // 先取消任何自动模式
          id(zw101_reader).cancel_auto_mode();
          delay(200);

          // 然后进入休眠
          id(zw101_reader).enter_sleep_mode();
```

---

### 原因2: UART缓冲区有残留数据

**症状**: 之前的命令响应还在缓冲区中

**解决方案**: 清空UART缓冲区

修改 `enter_sleep_mode()` 添加缓冲区清理：

```cpp
// 在发送休眠命令前清空接收缓冲区
bool ZW101Component::enter_sleep_mode() {
  // 清空接收缓冲区
  while (available()) {
    read();
  }
  delay(50);  // 等待硬件缓冲区清空

  uint8_t packet[12];
  // ... 其余代码
}
```

---

### 原因3: 超时时间不够

**症状**: 模组需要更长时间处理休眠命令

**当前超时**: 400ms
**建议超时**: 800-1000ms

修改超时时间：

```cpp
// zw101.cpp:497
uint8_t resp_len = wait_for_response(response, 32, 1000);  // 从400改为1000
```

---

### 原因4: 搜索流程干扰

**症状**: `process_search()` 在后台持续运行，干扰休眠命令

**最可能的原因！**

当前代码的问题：
```cpp
void ZW101Component::loop() {
  // ...

  // 处理搜索流程
  process_search();  // 这个每次loop都会运行！
}
```

即使你发送了休眠命令，`process_search()` 仍在每次 loop 中发送搜索命令，导致：
1. UART 被搜索命令占用
2. 模组无法正确响应休眠命令
3. 或模组刚进入休眠就被搜索命令唤醒

**解决方案A: 临时禁用搜索**

修改代码，在发送休眠命令前先设置标志：

```cpp
bool ZW101Component::enter_sleep_mode() {
  // 先设置休眠标志，停止搜索
  sleep_mode_ = true;

  // 等待当前搜索完成
  delay(100);

  // 清空缓冲区
  while (available()) {
    read();
  }

  // 发送休眠命令
  uint8_t packet[12];
  // ...
}
```

---

### 原因5: 校验和计算错误

**验证数据包是否正确**

标准休眠数据包应该是：
```
EF 01 FF FF FF FF 01 00 03 33 00 37
```

分解：
- `EF 01` - 包头
- `FF FF FF FF` - 设备地址
- `01` - 包标识
- `00 03` - 长度 (3字节)
- `33` - 指令码 (CMD_INTO_SLEEP)
- `00 37` - 校验和

校验和计算：
```
checksum = 0x01 + 0x03 + 0x33 = 0x37
```

当前代码：
```cpp
uint16_t checksum = 1 + length + CMD_INTO_SLEEP;
// checksum = 1 + 3 + 0x33 = 0x37 ✅ 正确
```

---

## 推荐修复方案

### 方案1: 修改 enter_sleep_mode 函数（推荐）

```cpp
// zw101.cpp
bool ZW101Component::enter_sleep_mode() {
  ESP_LOGI(TAG, "Preparing to enter sleep mode...");

  // 1. 先设置休眠标志，停止process_search()
  sleep_mode_ = true;

  // 2. 等待当前操作完成
  delay(200);

  // 3. 清空UART接收缓冲区
  while (available()) {
    read();
  }

  // 4. 构建休眠数据包
  uint8_t packet[12];
  uint16_t length = 3;
  uint16_t checksum = 1 + length + CMD_INTO_SLEEP;

  build_packet_header(packet, length);
  packet[9] = CMD_INTO_SLEEP;
  packet[10] = (checksum >> 8) & 0xFF;
  packet[11] = checksum & 0xFF;

  ESP_LOGI(TAG, "Sending sleep command...");
  ESP_LOG_BUFFER_HEX(TAG, packet, 12);

  // 5. 发送命令
  write_array(packet, 12);
  flush();

  // 6. 等待响应（增加超时时间）
  uint8_t response[32];
  uint8_t resp_len = wait_for_response(response, 32, 1000);  // 增加到1000ms

  ESP_LOGI(TAG, "Sleep response length: %d", resp_len);
  if (resp_len > 0) {
    ESP_LOG_BUFFER_HEX(TAG, response, resp_len);
  }

  if (resp_len >= 12 && response[9] == 0x00) {
    ESP_LOGI(TAG, "Module entered sleep mode successfully");
    if (status_sensor_) {
      status_sensor_->publish_state("Sleep Mode");
    }
    return true;
  }

  // 7. 失败时清除休眠标志
  sleep_mode_ = false;

  if (resp_len > 0 && resp_len >= 10) {
    ESP_LOGW(TAG, "Failed to enter sleep mode - Error code: 0x%02X", response[9]);
  } else {
    ESP_LOGW(TAG, "Failed to enter sleep mode - No response or timeout");
  }
  return false;
}
```

---

### 方案2: 简化方案 - 不用休眠，直接停止搜索

如果休眠命令一直失败，可以采用简化方案：

```cpp
// zw101.h 添加公共方法
public:
  void disable_auto_search() {
    sleep_mode_ = true;  // 利用这个标志停止搜索
    ESP_LOGI(TAG, "Auto search disabled");
  }

  void enable_auto_search() {
    sleep_mode_ = false;
    ESP_LOGI(TAG, "Auto search enabled");
  }
```

```yaml
# YAML 配置
button:
  - platform: template
    name: "Stop Search and Set LED"
    on_press:
      - lambda: |-
          // 直接停止搜索（不发送休眠命令）
          id(zw101_reader).disable_auto_search();
          delay(100);
          // 设置LED
          id(zw101_reader).set_rgb_led(3, 4, 150);
```

**优点**:
- ✅ 不依赖模组的休眠功能
- ✅ 100%可靠
- ✅ LED 可以持久保持
- ⚠️ 模组仍然耗电（不会真正休眠）

---

### 方案3: 重试机制

```cpp
bool ZW101Component::enter_sleep_mode() {
  const int MAX_RETRIES = 3;

  for (int retry = 0; retry < MAX_RETRIES; retry++) {
    if (retry > 0) {
      ESP_LOGW(TAG, "Retry entering sleep mode (%d/%d)", retry + 1, MAX_RETRIES);
      delay(500);
    }

    // 清空缓冲区
    while (available()) {
      read();
    }

    // 发送休眠命令...
    // （原有代码）

    if (成功) {
      return true;
    }
  }

  ESP_LOGE(TAG, "Failed to enter sleep mode after %d retries", MAX_RETRIES);
  return false;
}
```

---

## 调试步骤

重新编译并查看日志，现在应该会显示：

```
[I][zw101]: Sending sleep command...
[I][zw101]: EF 01 FF FF FF FF 01 00 03 33 00 37
[I][zw101]: Sleep response length: 12
[I][zw101]: EF 01 FF FF FF FF 07 00 03 00 XX XX
```

或者（失败）：

```
[I][zw101]: Sending sleep command...
[I][zw101]: EF 01 FF FF FF FF 01 00 03 33 00 37
[I][zw101]: Sleep response length: 0
[W][zw101]: Failed to enter sleep mode - No response or timeout
```

---

## 快速测试

### 测试1: 验证数据包

编译后查看日志中的 "Sending sleep command" 行，应该显示：
```
EF 01 FF FF FF FF 01 00 03 33 00 37
```

如果不一致，说明数据包构建有问题。

### 测试2: 验证响应

查看 "Sleep response length" 和后面的十六进制数据。

**成功响应**:
```
EF 01 FF FF FF FF 07 00 03 00 00 0B
                              ^^
                           确认码 0x00
```

**失败响应**:
```
响应长度: 0  → 超时，模组没响应
响应长度: > 0 但确认码 != 0x00 → 模组拒绝（可能正在执行其他操作）
```

---

## 最终建议

1. **立即可用**: 使用**方案2（简化方案）**，直接设置 `sleep_mode_ = true` 停止搜索
2. **长期方案**: 使用**方案1（修复休眠函数）**，完整实现休眠功能
3. **如果还失败**: 检查硬件连接、UART波特率、TX/RX引脚是否正确

---

**下一步**: 重新编译并查看调试日志，根据输出判断具体原因。
