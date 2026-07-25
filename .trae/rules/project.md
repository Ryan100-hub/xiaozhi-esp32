# XiaoZhi ESP32 项目规则

## 项目身份

- **名称**：XiaoZhi (小智) ESP32 语音助手固件
- **技术栈**：ESP-IDF C/C++，FreeRTOS
- **SDK 版本**：ESP-IDF v6.0.2（首选），v5.5.x 仅用于文档化的遗留开发板
- **规模**：138+ 开发板，172+ 变体，6 种 ESP32 芯片

## 核心原则

### 1. 单一开发板原则

- 每个构建必须通过 `DECLARE_BOARD(...)` 导出恰好一个开发板工厂
- 核心代码只依赖 `Board` 接口，从不依赖具体开发板类或 `config.h`
- 开发板身份影响 OTA 兼容性，永远不要修改现有开发板的引脚来支持不同硬件

### 2. 状态管理原则

- 只能通过 `Application::SetDeviceState()` 和 `DeviceStateMachine` 改变运行时状态
- 回调可能在主任务之外运行，使用 `Application::Schedule()` 或事件位调度应用变更
- 不要阻塞主事件循环或音频任务

### 3. 并发和性能原则

- 避免音频路径中的无界队列和重复大分配
- 使用专用 FreeRTOS 任务处理音频输入/输出/编解码
- 音频路径中的内存分配必须谨慎

### 4. 协议中性原则

- 保持共享消息语义在 `Protocol` 中
- 改变协议契约时必须验证 WebSocket 和 MQTT/UDP 两种传输
- 验证网络输入，保持 `cJSON` 所有权规则

### 5. 可选能力原则

- 摄像头、背光、显示、LED、电池等视为可选能力
- 不要假设每个目标都有 PSRAM 或 S3/P4 资源
- 使用 Kconfig/component 规则保护目标特定功能

## 禁止操作

### 禁止手动编辑的文件/目录

- `build/` - 构建输出
- `releases/` - 发布输出
- `managed_components/` - 管理的组件
- `components/` - 第三方组件
- `sdkconfig*` - SDK 配置文件
- `main/assets/lang_config.h` - 语言配置头文件
- 生成的 mmap 头文件

### 禁止的代码行为

- 不要假设所有目标都有 PSRAM
- 不要在不了解影响的情况下修改开发板引脚定义
- 不要在核心模块中放置开发板特定行为
- 不要进行无关的大规模代码格式化

## 构建命令

```sh
# 环境准备（必须）
source /path/to/esp-idf/export.sh

# 列出所有开发板
python3 scripts/release.py --list-boards

# 构建单个变体
python3 scripts/release.py <board-directory> --name <variant-name>

# 主机测试
python3 -m unittest discover -s scripts/tests -v

# 格式化（只格式化修改的文件）
clang-format -i <files>
```

## 测试要求

### 开发板相关变更

- 构建受影响的变体
- 烟雾测试改变的硬件

### 核心/通用变更

- 运行主机测试
- 构建代表性受影响的芯片/网络路径

### 协议变更

- 验证 WebSocket 和 MQTT/UDP（当共享行为改变时）

### 音频变更

- 验证采集、播放、唤醒/VAD、中断、重连、AEC 模式

### 重要

**成功的构建不是硬件验证**——始终报告测试了什么，什么仍需要物理硬件。

## 代码风格

- Google C++ 风格
- 4 空格缩进
- 100 列宽
- 使用仓库的 `.clang-format` 文件
- 只格式化修改的文件，避免无关格式化

## 关键文件

- `TRAEE.md` - 项目上下文文档
- `AGENTS.md` - 详尽规则
- `CLAUDE.md` - Claude Code 指南
- `docs/custom-board.md` - 开发板创建指南
- `main/audio/README.md` - 音频子系统设计
