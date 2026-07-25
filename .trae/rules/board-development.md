# 开发板开发指南

## 开发板开发流程

### 1. 准备工作

- 阅读 `docs/custom-board.md`
- 查看最接近的现有实现
- 理解硬件规格（芯片、引脚、外设）

### 2. 创建开发板目录

```
main/boards/<board-name>/
├── config.json          # 开发板配置（必需）
├── config.h             # 引脚和硬件定义（必需）
├── <board-name>.cc      # Board 实现（必需）
├── README.md            # 开发板文档（必需）
└── [其他文件]           # 可选的自定义编解码器、显示等
```

### 3. 创建 config.json

```json
{
  "target": "esp32s3",
  "builds": [
    {
      "name": "default",
      "sdkconfig_defaults": {
        "CONFIG_ESP32S3_DEFAULT_CPU_FREQ_240": true,
        "CONFIG_PARTITION_TABLE_CUSTOM_FILENAME": "partitions.csv"
      }
    }
  ]
}
```

**关键字段**：
- `target` - 目标芯片（esp32, esp32s3, esp32c3, esp32c5, esp32c6, esp32p4）
- `builds` - 构建变体数组
- `sdkconfig_defaults` - sdkconfig 覆盖

### 4. 创建 config.h

```cpp
#pragma once

// 引脚定义
#define PIN_I2S_BCK GPIO_NUM_XX
#define PIN_I2S_WS GPIO_NUM_XX
#define PIN_I2S_DOUT GPIO_NUM_XX
#define PIN_I2S_DIN GPIO_NUM_XX

// 网络类型
#define NETWORK_TYPE WiFi

// 可选外设
// #define HAS_DISPLAY
// #define HAS_CAMERA
// #define HAS_LED
```

### 5. 实现 Board 类

```cpp
#include "board.h"
#include "config.h"

class MyBoard : public Board {
public:
    std::string GetBoardType() override {
        return "my-board";
    }
    
    AudioCodec* GetAudioCodec() override {
        static MyAudioCodec codec;
        return &codec;
    }
    
    NetworkInterface* GetNetwork() override {
        static WiFiNetwork network;
        return &network;
    }
    
    void StartNetwork() override {
        // 启动网络
    }
    
    const char* GetNetworkStateIcon() override {
        return "wifi_icon";
    }
    
    // 可选：覆盖其他虚函数
    Display* GetDisplay() override { return nullptr; }
    Led* GetLed() override { return nullptr; }
    Camera* GetCamera() override { return nullptr; }
};

// 必须使用此宏声明开发板
DECLARE_BOARD(MyBoard);
```

### 6. 更新 Kconfig

在 `main/Kconfig.projbuild` 中添加：

```kconfig
config BOARD_TYPE_MY_BOARD
    bool "My Board Name"
    depends on SOC_ESP32S3_SUPPORTED
    help
        Select this board for custom hardware.
```

### 7. 更新 CMakeLists.txt

在 `main/CMakeLists.txt` 中添加映射：

```cmake
elseif(CONFIG_BOARD_TYPE_MY_BOARD)
    set(BOARD_TYPE "my-board")
    set(BOARD_DIR "my-board")
```

### 8. 创建 README.md

包含：
- 硬件规格
- 引脚图
- 构建命令
- 使用说明
- 图片（可选）

## 开发板变体

### 多变体开发板

一个开发板可以有多个变体（例如不同显示屏、不同网络配置）：

```json
{
  "target": "esp32s3",
  "builds": [
    {
      "name": "oled",
      "sdkconfig_defaults": {
        "CONFIG_BOARD_DISPLAY_OLED": true
      }
    },
    {
      "name": "lcd",
      "sdkconfig_defaults": {
        "CONFIG_BOARD_DISPLAY_LCD": true
      }
    }
  ]
}
```

### 变体命名规范

- 使用小写字母和连字符
- 描述主要差异（如 `oled`、`lcd`、`wifi`、`4g`）
- 保持简洁明了

## 常见开发板模式

### WiFi 开发板

```cpp
#include "boards/common/wifi_board.h"

class MyWiFiBoard : public WiFiBoard {
    // WiFiBoard 已实现大部分网络逻辑
    // 只需实现硬件特定部分
};
```

### 以太网开发板

```cpp
#include "boards/common/ethernet_board.h"

class MyEthernetBoard : public EthernetBoard {
    // EthernetBoard 提供以太网支持
};
```

### 4G 开发板

```cpp
#include "boards/common/ml307_board.h"
// 或
#include "boards/common/nt26_board.h"

class My4GBoard : public ML307Board {
    // ML307Board 提供 4G 支持
};
```

### 双网络开发板

```cpp
#include "boards/common/dual_network_board.h"

class MyDualNetworkBoard : public DualNetworkBoard {
    // 支持 WiFi + 4G 切换
};
```

## 自定义音频编解码器

### 简单编解码器

```cpp
#include "audio/audio_codec.h"

class MyAudioCodec : public AudioCodec {
public:
    MyAudioCodec() {
        // 设置基本参数
        input_sample_rate_ = 16000;
        output_sample_rate_ = 16000;
        input_channels_ = 1;
        output_channels_ = 1;
        duplex_ = false;  // 半双工
    }
    
protected:
    int Read(int16_t* dest, int samples) override {
        // 从 I2S 读取数据
        size_t bytes_read = 0;
        i2s_read(rx_handle_, dest, samples * 2, &bytes_read, portMAX_DELAY);
        return bytes_read / 2;
    }
    
    int Write(const int16_t* data, int samples) override {
        // 写入 I2S
        size_t bytes_written = 0;
        i2s_write(tx_handle_, data, samples * 2, &bytes_written, portMAX_DELAY);
        return bytes_written / 2;
    }
};
```

### 全双工编解码器（带 AEC）

```cpp
class MyDuplexCodec : public AudioCodec {
public:
    MyDuplexCodec() {
        duplex_ = true;           // 全双工
        input_reference_ = true;  // 需要参考信号用于 AEC
    }
};
```

## 自定义显示

### OLED 显示

```cpp
#include "display/oled_display.h"

class MyOLEDDisplay : public OLEDDisplay {
    // OLEDDisplay 已实现基本逻辑
    // 只需配置引脚和初始化
};
```

### LCD 显示（LVGL）

```cpp
#include "display/lvgl_display.h"

class MyLCDDisplay : public LVGLDisplay {
    // LVGLDisplay 提供 LVGL 集成
};
```

## 测试开发板

### 构建测试

```sh
# 构建开发板的所有变体
python3 scripts/release.py <board-name>

# 构建特定变体
python3 scripts/release.py <board-name> --name <variant-name>
```

### 烟雾测试

1. 烧录固件
2. 验证启动日志
3. 测试网络连接
4. 测试音频采集和播放
5. 测试唤醒词检测
6. 测试显示（如果有）
7. 测试 LED（如果有）

### 验证清单

- [ ] 开发板正确识别
- [ ] 网络连接正常
- [ ] 音频采集正常
- [ ] 音频播放正常
- [ ] 唤醒词检测正常
- [ ] 显示正常（如果有）
- [ ] LED 正常（如果有）
- [ ] 电池监控正常（如果有）
- [ ] OTA 更新正常

## 常见问题

### Q: 如何选择最接近的现有开发板？

A: 考虑以下因素：
- 相同芯片
- 相同网络类型
- 相同音频编解码器
- 相同显示类型

### Q: 可以修改现有开发板的引脚吗？

A: **不可以**。开发板身份影响 OTA 兼容性。如果需要不同引脚，添加新开发板或变体。

### Q: 如何处理可选外设？

A: 
- 在 `config.h` 中定义宏（如 `HAS_DISPLAY`）
- 在 Board 实现中检查宏并返回相应实例或 `nullptr`
- 核心代码会检查空指针

### Q: 如何添加自定义 MCP 工具？

A: 
1. 在开发板初始化时创建 `McpTool` 实例
2. 使用 `McpServer::GetInstance().AddTool()` 注册
3. 工具会在下次 MCP 请求时可用

### Q: 如何支持多种音频引擎？

A: 
- S3/P4 芯片使用 `AfeAudioEngine`（支持 AEC）
- C3/C5/C6 芯片使用 `LiteAudioEngine`
- 在 `GetAudioCodec()` 中根据芯片类型返回相应引擎

## 最佳实践

1. **先阅读现有实现** - 不要从零开始
2. **复用通用组件** - 使用 `boards/common/` 中的基础设施
3. **保持简洁** - 只实现必要的功能
4. **完整文档** - 包含硬件规格、构建命令、使用说明
5. **充分测试** - 构建测试 + 硬件测试
6. **遵循规范** - 代码风格、命名规范、目录结构
