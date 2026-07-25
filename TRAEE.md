# XiaoZhi ESP32 - Trae AI 上下文文档

## 项目概述

XiaoZhi (小智) 是一个基于 ESP-IDF 的 C/C++ 语音助手固件项目，支持 ESP32 系列芯片（ESP32、ESP32-C3/C5/C6、ESP32-S3、ESP32-P4）。

### 核心特性

- **多网络支持**：WiFi、以太网、USB RNDIS、ML307/EC801E/NT26 4G
- **离线语音唤醒**：基于 ESP-SR，支持自定义唤醒词
- **双传输协议**：WebSocket 和 MQTT+UDP
- **音频流**：Opus 编码，支持 ASR/LLM/TTS 管道和实时端到端语音模型
- **多语言**：38 种界面语言
- **MCP 协议**：设备端和云端 MCP 支持
- **多显示支持**：OLED、LCD、摄像头视觉输入

### 规模

- 138+ 开发板目录
- 172+ 发布变体
- 支持 6 种 ESP32 芯片平台

## 核心架构

### 主要组件

```
main/
├── application.cc/.h          # 主事件循环、协议生命周期、状态编排
├── device_state_machine.cc/.h # 状态转换验证和强制执行
├── device_state.h             # 设备状态枚举
├── mcp_server.cc/.h           # 设备端 MCP 工具注册和分发
├── settings.cc/.h             # NVS 持久化设置
├── ota.cc/.h                  # OTA 固件更新
├── assets.cc/.h               # 资源分区管理
├── audio/
│   ├── audio_service.cc/.h    # 双任务音频管道
│   ├── audio_codec.h          # 抽象硬件编解码器接口
│   ├── audio_engine.h         # 抽象唤醒词/VAD 引擎
│   ├── codecs/                # 具体编解码器实现
│   └── engines/               # AFE (S3/P4) 和 lite (C3/C5/C6) 引擎
├── protocols/
│   ├── protocol.h             # 抽象传输层接口
│   ├── websocket_protocol.cc/.h
│   └── mqtt_protocol.cc/.h
├── display/                   # OLED、LCD、表情显示
├── led/                       # LED 控制
└── boards/
    ├── common/                # 共享基础设施
    └── <board-name>/          # 具体开发板实现
```

### 关键设计模式

#### 1. Board 抽象

`Board` 是抽象基类（`main/boards/common/board.h`），每个构建通过 `DECLARE_BOARD(ClassName)` 选择一个具体实现。

```cpp
// 所有核心代码依赖 Board 接口，从不依赖具体类
class Board {
public:
    virtual std::string GetBoardType() = 0;
    virtual AudioCodec* GetAudioCodec() = 0;
    virtual NetworkInterface* GetNetwork() = 0;
    virtual void StartNetwork() = 0;
    // ... 其他虚函数
};

// 开发板实现使用宏声明
DECLARE_BOARD(MyBoardClass);
```

#### 2. 状态机

`DeviceStateMachine` 验证和强制执行合法状态转换：

```cpp
enum DeviceState {
    kDeviceStateUnknown,
    kDeviceStateStarting,
    kDeviceStateWifiConfiguring,
    kDeviceStateIdle,
    kDeviceStateConnecting,
    kDeviceStateListening,
    kDeviceStateSpeaking,
    kDeviceStateUpgrading,
    kDeviceStateActivating,
    kDeviceStateAudioTesting,
    kDeviceStateFatalError
};

// 只能通过 Application::SetDeviceState() 改变状态
```

#### 3. 音频数据流

```
MIC → AudioEngine (唤醒词/VAD/AEC) 
    → Opus 编码器 → 发送队列 → 服务器
    
服务器 → 解码队列 → Opus 解码器 
      → 播放队列 → 扬声器
```

专用 FreeRTOS 任务：输入任务、输出任务、Opus 编解码任务。

#### 4. 协议抽象

`Protocol` 是传输中性接口，WebSocket 和 MQTT/UDP 都实现它：

```cpp
class Protocol {
public:
    virtual bool OpenAudioChannel() = 0;
    virtual bool SendAudio(std::unique_ptr<AudioStreamPacket> packet) = 0;
    virtual void SendText(const std::string& text) = 0;
    // 回调注册
    void OnIncomingAudio(std::function<void(...)> callback);
    void OnIncomingJson(std::function<void(...)> callback);
};
```

## 开发板选择链

添加开发板需要更新整个链条：

1. **config.json**（开发板目录）- 定义目标芯片、构建变体、sdkconfig 覆盖
2. **scripts/release.py** - 通过 config.json 发现开发板，驱动构建
3. **main/Kconfig.projbuild** - Kconfig `CONFIG_BOARD_TYPE_*` 符号
4. **main/CMakeLists.txt** - 映射 `CONFIG_BOARD_TYPE_*` 到 `BOARD_TYPE` 字符串
5. **开发板源文件** - `config.h`、实现 `Board` 的 `.cc` 文件、`DECLARE_BOARD`
6. **开发板文档** - 开发板目录中的 `README.md`

**重要**：开发板身份影响 OTA 兼容性。永远不要修改现有开发板的引脚来支持不同硬件——添加新开发板或变体。

## 构建命令

### 环境准备

```sh
# 必须先 source ESP-IDF 环境
source /path/to/esp-idf/export.sh
```

### 常用命令

```sh
# 列出所有开发板变体
python3 scripts/release.py --list-boards

# 构建单个变体（规范入口点；会改变 sdkconfig）
python3 scripts/release.py <board-directory> --name <variant-name>

# 构建开发板的所有变体
python3 scripts/release.py <board-directory>

# 主机端测试
python3 -m unittest discover -s scripts/tests -v

# 格式化文件（Google C++ 风格，4 空格缩进，100 列宽）
clang-format -i <file>
clang-format --dry-run -Werror <file>
```

**注意**：release.py 脚本会改变本地 `sdkconfig` 和构建状态。不要假设构建目录仍代表之前的目标。

## 关键规则

### 状态管理

- 只能通过 `Application::SetDeviceState()` 和 `DeviceStateMachine` 改变运行时状态
- 回调可能在主任务之外运行。使用 `Application::Schedule()` 或事件位调度应用变更

### 并发和性能

- 不要阻塞主事件循环或音频任务
- 避免音频路径中的无界队列和重复大分配
- 使用专用 FreeRTOS 任务处理音频输入/输出/编解码

### 协议

- 保持共享消息语义在 `Protocol` 中
- 改变协议契约时验证 WebSocket 和 MQTT/UDP 两种传输

### 网络和数据

- 验证网络输入
- 保持 `cJSON` 所有权规则
- NVS 键是持久化 API，改变时需要迁移

### 目标特定功能

- 使用 Kconfig/component 规则保护目标特定功能
- 不要假设每个目标都有 PSRAM 或 S3/P4 资源

### 可选能力

- 摄像头、背光、显示、LED、电池等视为可选能力
- 核心代码不能假设这些能力存在

### 代码格式

- 只格式化修改的 C/C++ 文件，使用仓库的 `.clang-format`
- 避免无关的大规模格式化

### 禁止手动编辑

以下文件/目录禁止手动编辑：
- `build/`
- `releases/`
- `managed_components/`
- `components/`
- `sdkconfig*`
- `main/assets/lang_config.h`
- 生成的 mmap 头文件

## 测试和验证

### 开发板相关变更

- 构建受影响的变体
- 烟雾测试改变的硬件

### 核心/通用/音频/协议/显示/Kconfig/CMake 变更

- 运行主机测试（`scripts/tests/`）
- 构建代表性受影响的芯片/网络路径

### 协议变更

- 验证 WebSocket 和 MQTT/UDP（当共享行为改变时）

### 音频变更

- 验证采集、播放、唤醒/VAD、中断、重连、适用的 AEC 模式

### 重要原则

**成功的构建不是硬件验证**——始终报告测试了什么，什么仍需要物理硬件。

## 权威文档

- `AGENTS.md` - 详尽的规则、约定和子系统细节
- `CLAUDE.md` - Claude Code 指南（与本文件互补）
- `docs/esp-idf-6-migration.md` - SDK 兼容性和迁移说明
- `docs/custom-board.md` - 自定义开发板创建指南
- `docs/websocket.md` / `docs/mqtt-udp.md` - 线路协议细节
- `docs/mcp-protocol.md` / `docs/mcp-usage.md` - MCP 设备控制
- `main/audio/README.md` - 音频子系统设计
- `docs/code_style.md` - 代码格式化指南
- `.github/workflows/build.yml` - CI 构建矩阵

## 开发工作流

### 添加新开发板

1. 阅读 `docs/custom-board.md`
2. 查看最接近的现有实现
3. 创建开发板目录和 `config.json`
4. 实现 `Board` 子类，使用 `DECLARE_BOARD`
5. 更新 `main/Kconfig.projbuild`
6. 更新 `main/CMakeLists.txt`
7. 创建开发板 `README.md`
8. 构建并测试

### 修改核心代码

1. 理解对 `Board` 接口的影响
2. 检查是否影响 WebSocket 和 MQTT/UDP
3. 运行主机测试
4. 构建代表性变体
5. 报告测试覆盖范围

### 修改音频代码

1. 理解对采集、播放、唤醒/VAD、中断、重连、AEC 的影响
2. 考虑不同芯片平台（S3/P4 vs C3/C5/C6）的差异
3. 构建并测试受影响的变体
4. 报告需要物理硬件验证的部分

## 常见问题

### Q: 如何知道我的修改影响了哪些开发板？

A: 使用 `python3 scripts/release.py --list-boards` 查看所有开发板。核心代码修改通常影响所有开发板，开发板特定修改只影响该开发板。

### Q: 如何测试我的修改？

A: 
- 主机测试：`python3 -m unittest discover -s scripts/tests -v`
- 构建测试：使用 `scripts/release.py` 构建代表性变体
- 硬件测试：需要物理设备验证

### Q: 如何添加新的音频编解码器？

A: 
1. 在 `main/audio/codecs/` 创建新文件
2. 继承 `AudioCodec` 类
3. 实现 `Read()` 和 `Write()` 虚函数
4. 在开发板的 `GetAudioCodec()` 中返回实例

### Q: 如何添加新的 MCP 工具？

A: 
1. 创建 `McpTool` 实例
2. 使用 `McpServer::GetInstance().AddTool()` 注册
3. 工具会在下次 MCP 请求时可用

## 技术栈

- **语言**：C/C++（ESP-IDF 框架）
- **构建系统**：CMake + ESP-IDF 构建系统
- **RTOS**：FreeRTOS
- **网络**：WiFi、以太网、4G（ML307/EC801E/NT26）
- **协议**：WebSocket、MQTT+UDP
- **音频**：Opus 编解码、ESP-SR（唤醒词/VAD/AEC）
- **显示**：LVGL、OLED（SSD1306）
- **存储**：NVS（Non-Volatile Storage）
- **目标芯片**：ESP32、ESP32-C3/C5/C6、ESP32-S3、ESP32-P4
- **SDK 版本**：ESP-IDF v6.0.2（首选），v5.5.x（仅用于文档化的遗留开发板）

## 代码风格

- Google C++ 风格
- 4 空格缩进
- 100 列宽
- 使用仓库的 `.clang-format` 文件
- 只格式化修改的文件

## 版本控制

- 主分支 targeting ESP-IDF v6.0+
- v6.0.2 是首选稳定 SDK
- ESP-IDF v5.5 仅用于文档化的遗留开发板
- 当前矩阵包含 172 个变体

## 许可证

MIT 许可证，允许任何人免费使用，包括商业用途。
