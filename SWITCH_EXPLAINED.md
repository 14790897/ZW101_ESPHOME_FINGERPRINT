# switch.py 详细解释

## 文件作用

`switch.py` 是 ZW101 组件的 **Switch 平台** Python 代码,负责:
1. **YAML 配置验证** - 检查用户配置是否正确
2. **代码生成** - 生成 C++ 代码连接 YAML 配置和 C++ 实现

## 完整代码解析

### 1. 导入模块 (第 1-7 行)

```python
"""ZW101 Switch 平台"""
import esphome.codegen as cg              # 代码生成器
import esphome.config_validation as cv    # 配置验证器
from esphome.components import switch     # Switch 基础组件
from esphome.const import CONF_ID         # 标准常量

from . import ZW101Component, zw101_ns    # 从 __init__.py 导入
```

**说明**:
- `cg` - 生成 C++ 代码的工具
- `cv` - 验证 YAML 配置的工具
- `switch` - ESPHome 内置的 Switch 平台
- `ZW101Component` - 主组件类
- `zw101_ns` - C++ 命名空间 `esphome::zw101`

### 2. 依赖声明 (第 9 行)

```python
DEPENDENCIES = ["zw101"]
```

**说明**: 这个 Switch 平台依赖 `zw101` 主组件,必须先配置主组件才能使用 Switch。

### 3. 配置常量定义 (第 11-13 行)

```python
CONF_ZW101_ID = "zw101_id"  # 主组件 ID 配置项
CONF_ENROLL = "enroll"      # 注册开关配置项
CONF_CLEAR = "clear"        # 清空开关配置项
```

**对应的 YAML 配置**:
```yaml
switch:
  - platform: zw101
    zw101_id: zw101_reader    # <- CONF_ZW101_ID
    enroll:                   # <- CONF_ENROLL
      name: "Enroll Fingerprint"
    clear:                    # <- CONF_CLEAR
      name: "Clear Library"
```

### 4. C++ 类声明 (第 15-16 行)

```python
EnrollSwitch = zw101_ns.class_("EnrollSwitch", switch.Switch, cg.Component)
ClearSwitch = zw101_ns.class_("ClearSwitch", switch.Switch, cg.Component)
```

**等价于 C++ 代码**:
```cpp
namespace esphome {
namespace zw101 {
  class EnrollSwitch : public switch_::Switch, public Component { };
  class ClearSwitch : public switch_::Switch, public Component { };
}
}
```

**说明**:
- `zw101_ns.class_()` - 在 `zw101` 命名空间中声明类
- `switch.Switch` - 继承 ESPHome 的 Switch 基类
- `cg.Component` - 继承 Component 基类(可被 ESPHome 管理)

### 5. 配置模式定义 (第 18-30 行)

```python
CONFIG_SCHEMA = cv.Schema(
    {
        cv.GenerateID(CONF_ZW101_ID): cv.use_id(ZW101Component),
        cv.Optional(CONF_ENROLL): switch.switch_schema(
            EnrollSwitch,
            icon="mdi:fingerprint-add",
        ),
        cv.Optional(CONF_CLEAR): switch.switch_schema(
            ClearSwitch,
            icon="mdi:delete-forever",
        ),
    }
)
```

**逐行解释**:

#### 第 20 行: 引用主组件
```python
cv.GenerateID(CONF_ZW101_ID): cv.use_id(ZW101Component),
```
- `cv.GenerateID()` - 自动生成或使用指定的 ID
- `cv.use_id(ZW101Component)` - 引用已存在的 ZW101Component 实例
- **作用**: 连接到主组件,可以调用主组件的方法

#### 第 21-24 行: 注册开关配置
```python
cv.Optional(CONF_ENROLL): switch.switch_schema(
    EnrollSwitch,
    icon="mdi:fingerprint-add",
),
```
- `cv.Optional()` - 这个配置项是可选的
- `switch.switch_schema()` - 使用标准 Switch 配置模式
- `EnrollSwitch` - 使用的 C++ 类
- `icon` - Home Assistant 中显示的图标

#### 第 25-28 行: 清空开关配置
```python
cv.Optional(CONF_CLEAR): switch.switch_schema(
    ClearSwitch,
    icon="mdi:delete-forever",
),
```
同理,定义清空指纹库的开关。

### 6. 代码生成函数 (第 33-45 行)

```python
async def to_code(config):
    """生成 switch 代码"""
    # 获取主组件实例
    parent = await cg.get_variable(config[CONF_ZW101_ID])

    # 如果配置了 enroll 开关
    if CONF_ENROLL in config:
        sw = await switch.new_switch(config[CONF_ENROLL])
        cg.add(sw.set_parent(parent))
        cg.add(parent.set_enroll_switch(sw))

    # 如果配置了 clear 开关
    if CONF_CLEAR in config:
        sw = await switch.new_switch(config[CONF_CLEAR])
        cg.add(sw.set_parent(parent))
        cg.add(parent.set_clear_switch(sw))
```

**详细解释**:

#### 第 35 行: 获取主组件
```python
parent = await cg.get_variable(config[CONF_ZW101_ID])
```
- 获取之前创建的 `ZW101Component` 实例
- 通过 `zw101_id` 配置项指定

#### 第 37-40 行: 创建注册开关
```python
if CONF_ENROLL in config:
    sw = await switch.new_switch(config[CONF_ENROLL])
    cg.add(sw.set_parent(parent))
    cg.add(parent.set_enroll_switch(sw))
```

**生成的 C++ 代码**:
```cpp
auto *sw = new EnrollSwitch();
// ... 配置 name, icon 等 ...
sw->set_parent(parent);           // ← 设置父组件指针
parent->set_enroll_switch(sw);    // ← 主组件保存开关指针
```

**双向关联**:
```
EnrollSwitch  ←──→  ZW101Component
   (sw)              (parent)
```

#### 第 42-45 行: 创建清空开关
```python
if CONF_CLEAR in config:
    sw = await switch.new_switch(config[CONF_CLEAR])
    cg.add(sw.set_parent(parent))
    cg.add(parent.set_clear_switch(sw))
```
同理,创建清空开关并建立双向关联。

## 工作流程图

```
┌──────────────────┐
│  YAML 配置文件    │
│                  │
│  switch:         │
│    - platform: zw101
│      zw101_id: xxx
│      enroll:     │
│        name: ... │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  switch.py       │
│  (配置验证)       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  to_code()       │
│  (代码生成)       │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────┐
│  生成的 C++ 代码              │
│                              │
│  auto *sw = new EnrollSwitch();
│  sw->set_parent(parent);     │
│  parent->set_enroll_switch(sw);
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│  运行时 (C++)                 │
│                              │
│  用户按下开关                 │
│    ↓                         │
│  EnrollSwitch::write_state() │
│    ↓                         │
│  parent_->register_fingerprint()
│    ↓                         │
│  ZW101Component 执行注册      │
└──────────────────────────────┘
```

## C++ 对应实现

### EnrollSwitch 类 (zw101.h:109-122)

```cpp
class EnrollSwitch : public switch_::Switch, public Component {
 public:
  void set_parent(ZW101Component *parent) {
    parent_ = parent;  // 保存主组件指针
  }

 protected:
  void write_state(bool state) override {
    if (state && parent_) {          // 如果开关被打开
      parent_->register_fingerprint();  // 调用主组件的注册方法
      publish_state(false);          // 自动关闭开关
    }
  }

  ZW101Component *parent_{nullptr};  // 指向主组件的指针
};
```

**工作流程**:
1. Home Assistant 用户打开开关
2. `write_state(true)` 被调用
3. 调用 `parent_->register_fingerprint()`
4. 主组件开始非阻塞注册流程
5. 开关自动关闭 `publish_state(false)`

### ClearSwitch 类 (zw101.h:125-137)

```cpp
class ClearSwitch : public switch_::Switch, public Component {
 public:
  void set_parent(ZW101Component *parent) {
    parent_ = parent;
  }

 protected:
  void write_state(bool state) override {
    if (state && parent_) {                // 如果开关被打开
      parent_->clear_fingerprint_library();  // 调用清空方法
      publish_state(false);                // 自动关闭开关
    }
  }

  ZW101Component *parent_{nullptr};
};
```

## YAML 配置示例

```yaml
# 主组件配置
zw101:
  id: zw101_reader
  uart_id: fingerprint_uart

# Switch 平台配置
switch:
  - platform: zw101        # 使用 zw101 的 switch 平台
    zw101_id: zw101_reader # 引用主组件 ID

    enroll:                # 注册开关 (可选)
      name: "Enroll Fingerprint"
      icon: "mdi:fingerprint-add"  # 自动设置

    clear:                 # 清空开关 (可选)
      name: "Clear Library"
      icon: "mdi:delete-forever"   # 自动设置
```

## 为什么需要双向关联?

### Switch → Component (通过 `parent_`)
开关需要调用主组件的方法:
```cpp
parent_->register_fingerprint();
parent_->clear_fingerprint_library();
```

### Component → Switch (通过 `set_enroll_switch()`)
主组件可能需要控制开关状态(虽然当前未使用):
```cpp
if (enroll_switch_) {
  enroll_switch_->publish_state(true);  // 更新开关显示
}
```

## 关键设计模式

### 1. **Platform Pattern** (平台模式)
每种传感器/开关类型都是一个"平台":
- `binary_sensor.py` - Binary Sensor 平台
- `sensor.py` - Sensor 平台
- `switch.py` - Switch 平台
- `text_sensor.py` - Text Sensor 平台

### 2. **Dependency Injection** (依赖注入)
通过 `set_parent()` 注入主组件依赖:
```python
cg.add(sw.set_parent(parent))
```

### 3. **Auto-off Pattern** (自动关闭模式)
开关触发动作后自动关闭,避免重复触发:
```cpp
parent_->register_fingerprint();
publish_state(false);  // 立即关闭
```

## 总结

`switch.py` 的核心作用:

1. **配置验证** ✅
   - 检查 YAML 配置是否合法
   - 确保引用的主组件存在

2. **代码生成** ✅
   - 创建 C++ Switch 对象
   - 建立双向关联

3. **类型安全** ✅
   - 使用强类型的 Python 代码
   - 生成类型正确的 C++ 代码

4. **用户友好** ✅
   - 简洁的 YAML 配置
   - 自动设置图标和样式

这就是 ESPHome External Component 的 **Python 配置层** 和 **C++ 运行时层** 的协作方式!
