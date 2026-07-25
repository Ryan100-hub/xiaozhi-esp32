# 仓库指南

## 项目概述

小智（XiaoZhi）是一款基于 ESP-IDF 的 C/C++ 语音助手固件，支持 138+ 种板卡目录、172+ 个发布变体，覆盖 ESP32 / C3 / C5 / C6 / S3 / P4 芯片平台。通过 Qwen / DeepSeek 等大模型提供语音交互能力，基于 MCP 协议实现多端设备控制。首选 ESP-IDF 版本为 v6.0.2。

## 项目结构与模块组织

```
main/
├── application.*        —  主事件循环、协议生命周期、高层调度
├── device_state_machine.* —  运行时状态合法转换（11 种状态）
├── mcp_server.*         —  设备端 MCP 工具分发（音量、灯光、GPIO 等）
├── ota.* / settings.* / assets.* / system_info.* —  支撑模块
├── boards/
│   ├── common/          —  Board 抽象接口，WiFi/4G/以太网网络后端
│   ├── <厂商>/<板卡>/    —  各板卡引脚映射、config.h、config.json、DECLARE_BOARD
│   └── ...              —  138+ 个板卡目录
├── audio/
│   ├── codecs/          —  ES8311 / ES8374 / ES8388 / ES8389 / NoAudio / Dummy / Box 音频编解码器
│   ├── engines/         —  AFE（音频前端）与 Lite 引擎封装
│   ├── wake_words/      —  自定义 + ESP-SR 唤醒词检测
│   └── demuxer/         —  OGG 解复用器，用于音频流
├── protocols/           —  Protocol 基类，WebSocket + MQTT/UDP 传输层
├── display/             —  OLED、LCD、LVGL 显示后端；支持表情、GIF、JPG
├── led/                 —  单灯、环形灯带（WS2812）、GPIO LED 模式
└── assets/              —  多语言语音提示（38 种语言）、通用音效
```

```
scripts/
├── release.py           —  任意板卡/变体的标准构建入口
├── gen_lang.py          —  语言代码生成器
├── build_default_assets.py —  资源打包工具
└── tests/               —  宿主机端 Python 单元测试
```

## 架构与设计模式

**各组件职责：**

| 模块 | 职责 |
|---|---|
| `Application` | 单例；运行 FreeRTOS 主事件循环；分发 `MAIN_EVENT_*` 事件位 |
| `DeviceStateMachine` | 强制约束 `DeviceState` 之间的合法状态转换 |
| `Board` | 抽象工厂（`create_board()`）；对外暴露音频编解码器、显示屏、LED、摄像头、网络、电池接口 |
| `Protocol` | 抽象传输层；`OnIncomingAudio` / `OnIncomingJson` 回调链 |
| `MCP Server` | 注册云端 MCP 请求下发的工具处理函数 |

**设计规则：**

- 核心代码依赖 `Board` 接口，绝不依赖具体的板卡类或板卡 `config.h`。
- 仅通过 `Application::SetDeviceState()` 和状态机更改运行时状态。
- 回调可能在主任务之外触发；使用 `Application::Schedule()` 或事件位将操作调度回主任务。
- 绝不在主事件循环或音频任务中阻塞。音频路径中避免无界队列和大量重复内存分配。

## 构建、测试与开发命令

首先加载 ESP-IDF 环境：

```sh
source /path/to/esp-idf/export.sh
idf.py --version
```

**查看板卡并构建：**

```sh
# 列出所有已知板卡/变体名称
python3 scripts/release.py --list-boards

# 标准变体构建（会修改本地 sdkconfig）
python3 scripts/release.py <板卡目录> --name <变体名称>

# 宿主机端测试
python3 -m unittest discover -s scripts/tests -v
```

## 编码风格与命名规范

- C/C++ 文件使用仓库根目录的 `.clang-format`；仅格式化你修改过的文件。
- 文件命名：源码使用 `snake_case.cc` / `snake_case.h`；板卡实现文件以板卡目录名命名。
- 类命名：`PascalCase`（如 `Application`、`MqttProtocol`）。
- 常量：枚举值使用 `kPascalCase`（如 `kDeviceStateIdle`），预处理器宏使用 `UPPER_SNAKE_CASE`。
- 每个板卡必须通过 `DECLARE_BOARD(...)` 导出恰好一个工厂。
- 共享消息语义保持在 `Protocol` 中；修改其契约时需同时验证两种传输方式（WebSocket + MQTT/UDP）。

## 板卡与硬件配置

板卡选择是一个耦合的链条：

```
config.json -> scripts/release.py -> main/Kconfig.projbuild -> main/CMakeLists.txt -> 板卡源码 + config.h
```

添加板卡或变体时，需更新链条中的每个环节。包含：唯一的板卡标识、正确的 `idf_target`、Flash/分区设置、恰好一个 `DECLARE_BOARD`，以及板卡说明文档（`README.md`）。摄像头、背光、显示屏、LED、电池等均视为可选能力。

## 测试指南

- **仅板卡变更**：构建受影响板卡的所有变体；对变更硬件进行冒烟测试。
- **核心 / 音频 / 协议 / 显示变更**：运行宿主机端测试，并构建有代表性的芯片/网络路径。
- **协议变更**：同时验证 WebSocket 和 MQTT/UDP 两种传输方式。
- **音频变更**：验证采集、播放、唤醒/VAD、打断、重连等路径。
- 构建成功不等于硬件验证通过；需报告已测试的内容以及仍需实体硬件验证的部分。

## 提交与 Pull Request 指南

- Git 历史遵循 Conventional Commits 风格（`feat:`、`fix:`、`refactor:`、`docs:`、`chore:`）。
- PR 应包含变更说明、受影响的板卡/变体及构建验证结果。
- 绝不通过修改已有板卡的引脚来适配不同硬件；应新增唯一命名的板卡或发布变体。
- 不要手动编辑生成/第三方产物：`build/`、`releases/`、`managed_components/`、`sdkconfig*`，以及生成的语言/资源头文件。
