# 架构规则

## 核心组件职责

### Application（主应用）

- **位置**：`main/application.cc/.h`
- **职责**：主事件循环、协议生命周期、状态编排
- **模式**：单例模式（`Application::GetInstance()`）
- **关键方法**：
  - `Initialize()` - 初始化显示、音频、网络回调
  - `Run()` - 主事件循环（永不返回）
  - `SetDeviceState()` - 请求状态转换
  - `Schedule()` - 在主任务中调度回调

### DeviceStateMachine（状态机）

- **位置**：`main/device_state_machine.cc/.h`
- **职责**：验证和强制执行合法状态转换
- **模式**：观察者模式（状态变化监听）
- **状态枚举**：`main/device_state.h`

### Board（开发板抽象）

- **位置**：`main/boards/common/board.h`
- **职责**：硬件抽象层接口
- **模式**：抽象基类 + 工厂模式
- **关键接口**：
  - `GetAudioCodec()` - 音频编解码器
  - `GetNetwork()` - 网络接口
  - `GetDisplay()` - 显示（可选）
  - `GetLed()` - LED（可选）
  - `GetCamera()` - 摄像头（可选）

### AudioService（音频服务）

- **位置**：`main/audio/audio_service.cc/.h`
- **职责**：双任务音频管道
- **数据流**：
  ```
  MIC → AudioEngine → Opus 编码 → 发送队列 → 服务器
  服务器 → 解码队列 → Opus 解码 → 播放队列 → 扬声器
  ```
- **任务**：输入任务、输出任务、Opus 编解码任务

### Protocol（协议抽象）

- **位置**：`main/protocols/protocol.h`
- **职责**：传输中性接口
- **实现**：
  - `WebSocketProtocol` - WebSocket 传输
  - `MqttProtocol` - MQTT+UDP 传输
- **关键方法**：
  - `OpenAudioChannel()` - 打开音频通道
  - `SendAudio()` - 发送音频数据
  - `SendText()` - 发送文本消息

### McpServer（MCP 服务器）

- **位置**：`main/mcp_server.cc/.h`
- **职责**：设备端 MCP 工具注册和分发
- **模式**：单例模式
- **关键类**：
  - `McpTool` - 工具定义
  - `Property` - 工具属性
  - `ReturnValue` - 返回值类型

## 组件交互规则

### 状态转换流程

```
用户操作/事件 
  → Application::SetDeviceState()
  → DeviceStateMachine::TransitionTo()
  → 验证转换合法性
  → 通知监听器
  → Application::OnStateChanged()
```

### 音频数据流

```
硬件麦克风
  → AudioCodec::InputData()
  → AudioEngine::Feed()（唤醒词/VAD/AEC）
  → 编码队列
  → Opus 编码任务
  → 发送队列
  → Protocol::SendAudio()
  → 服务器

服务器
  → Protocol 回调
  → 解码队列
  → Opus 解码任务
  → 播放队列
  → AudioCodec::OutputData()
  → 硬件扬声器
```

### 协议消息流

```
服务器消息
  → Protocol 实现（WebSocket/MQTT）
  → 回调分发
  → Application 处理
  → 状态更新/UI 更新

设备消息
  → Application
  → Protocol::SendText()
  → Protocol 实现
  → 服务器
```

## 开发板选择链

添加开发板必须更新整个链条：

```
config.json（开发板目录）
  ↓
scripts/release.py（发现开发板）
  ↓
main/Kconfig.projbuild（CONFIG_BOARD_TYPE_*）
  ↓
main/CMakeLists.txt（映射到 BOARD_TYPE）
  ↓
开发板源文件（config.h + Board 实现 + DECLARE_BOARD）
  ↓
开发板文档（README.md）
```

## 依赖规则

### 核心代码依赖

- 核心代码只依赖 `Board` 接口
- 不依赖具体开发板类
- 不依赖开发板 `config.h`

### 可选能力依赖

- 显示、LED、摄像头、背光、电池视为可选
- 使用空指针检查或 Kconfig 保护
- 不要假设能力存在

### 目标特定依赖

- 使用 Kconfig/component 规则保护
- 不要假设 PSRAM 存在
- 不要假设 S3/P4 资源

## 回调和并发规则

### 回调上下文

- 回调可能在主任务之外运行
- 网络回调、音频回调、定时器回调都可能在不同任务中

### 线程安全

- 使用 `Application::Schedule()` 调度主任务变更
- 使用事件位（`EventGroupHandle_t`）进行任务同步
- 避免在回调中直接修改共享状态

### 阻塞规则

- 不要阻塞主事件循环
- 不要阻塞音频任务
- 避免在音频路径中使用互斥锁（除非必要）

## 内存管理规则

### 音频路径

- 避免重复大分配
- 使用预分配缓冲区
- 队列大小有上限（参见 `audio_service.h` 中的宏定义）

### 网络数据

- 验证 `cJSON` 所有权
- 及时释放不再使用的 JSON 对象
- 避免内存泄漏

### NVS 存储

- NVS 键是持久化 API
- 改变键名时需要迁移
- 使用 `Settings` 类封装访问

## 扩展点

### 添加新开发板

1. 创建开发板目录和 `config.json`
2. 实现 `Board` 子类
3. 使用 `DECLARE_BOARD` 宏
4. 更新 Kconfig 和 CMakeLists.txt
5. 创建文档

### 添加新音频编解码器

1. 继承 `AudioCodec` 类
2. 实现 `Read()` 和 `Write()` 虚函数
3. 在开发板中返回实例

### 添加新 MCP 工具

1. 创建 `McpTool` 实例
2. 使用 `McpServer::GetInstance().AddTool()` 注册
3. 工具会在下次 MCP 请求时可用

### 添加新协议

1. 继承 `Protocol` 类
2. 实现所有纯虚函数
3. 在 `Application::InitializeProtocol()` 中选择
