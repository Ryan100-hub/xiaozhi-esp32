# 音频系统规则

## 音频架构概述

### 核心组件

- **AudioService** - 音频服务主类（`main/audio/audio_service.cc/.h`）
- **AudioCodec** - 硬件编解码器抽象（`main/audio/audio_codec.h`）
- **AudioEngine** - 唤醒词/VAD/AEC 引擎抽象（`main/audio/audio_engine.h`）
- **具体编解码器** - `main/audio/codecs/` 目录
- **具体引擎** - `main/audio/engines/` 目录

### 数据流

```
采集路径：
MIC → AudioCodec::InputData()
    → AudioEngine::Feed()（唤醒词/VAD/AEC 处理）
    → 编码队列（audio_encode_queue_）
    → Opus 编码任务
    → 发送队列（audio_send_queue_）
    → Protocol::SendAudio()
    → 服务器

播放路径：
服务器 → Protocol 回调
       → 解码队列（audio_decode_queue_）
       → Opus 解码任务
       → 播放队列（audio_playback_queue_）
       → AudioCodec::OutputData()
       → 扬声器
```

## FreeRTOS 任务

### 三个专用任务

1. **音频输入任务**（`audio_input_task_handle_`）
   - 从 AudioCodec 读取数据
   - 送入 AudioEngine 处理
   - 将处理后的 PCM 送入编码队列

2. **音频输出任务**（`audio_output_task_handle_`）
   - 从播放队列读取 PCM
   - 写入 AudioCodec 播放

3. **Opus 编解码任务**（`opus_codec_task_handle_`）
   - 从编码队列读取 PCM，编码为 Opus，送入发送队列
   - 从解码队列读取 Opus，解码为 PCM，送入播放队列

### 任务优先级

- 输入任务：高优先级（实时采集）
- 输出任务：高优先级（实时播放）
- 编解码任务：中等优先级

## 队列和缓冲区

### 队列大小限制

```cpp
#define OPUS_FRAME_DURATION_MS 60
#define MAX_ENCODE_TASKS_IN_QUEUE 2
#define MAX_PLAYBACK_TASKS_IN_QUEUE 2
#define MAX_DECODE_PACKETS_IN_QUEUE (2400 / OPUS_FRAME_DURATION_MS)  // 40
#define MAX_SEND_PACKETS_IN_QUEUE (2400 / OPUS_FRAME_DURATION_MS)    // 40
```

### 帧大小

- Opus 帧时长：60ms
- 采样率：16kHz（采集）/ 24kHz（播放，可配置）
- 位深：16-bit

## AudioCodec 接口

### 基本参数

```cpp
class AudioCodec {
protected:
    bool duplex_;              // 是否全双工
    bool input_reference_;     // 是否需要参考信号（AEC）
    int input_sample_rate_;    // 采集采样率
    int output_sample_rate_;   // 播放采样率
    int input_channels_;       // 采集通道数
    int output_channels_;      // 播放通道数
    
    virtual int Read(int16_t* dest, int samples) = 0;
    virtual int Write(const int16_t* data, int samples) = 0;
};
```

### 实现要点

1. **I2S 配置**
   - 使用 ESP-IDF I2S 驱动
   - 配置 DMA 描述符和帧大小
   - 注意 IDF 5 和 IDF 6 的 API 差异

2. **采样率匹配**
   - 采集采样率通常为 16kHz（唤醒词识别要求）
   - 播放采样率通常为 24kHz（服务器配置）
   - 需要重采样时使用 `input_resampler_` 和 `output_resampler_`

3. **双工模式**
   - 半双工：`duplex_ = false`
   - 全双工：`duplex_ = true`，需要 AEC 支持

## AudioEngine 接口

### 两种实现

1. **AfeAudioEngine**（`main/audio/engines/afe_audio_engine.cc`）
   - 用于 ESP32-S3 和 ESP32-P4
   - 支持 AEC（回声消除）
   - 支持 VAD（语音活动检测）
   - 支持唤醒词检测
   - 需要较多内存

2. **LiteAudioEngine**（`main/audio/engines/lite_audio_engine.cc`）
   - 用于 ESP32-C3/C5/C6
   - 不支持 AEC
   - 支持 VAD 和唤醒词检测
   - 内存占用较少

### 关键方法

```cpp
class AudioEngine {
public:
    virtual bool Initialize(AudioCodec* codec, int frame_duration_ms, srmodel_list_t* models_list) = 0;
    virtual void Feed(std::vector<int16_t>&& data) = 0;
    
    virtual void EnableWakeWordDetection(bool enable) = 0;
    virtual void EnableVoiceProcessing(bool enable) = 0;
    virtual void EnableDeviceAec(bool enable) = 0;
    
    virtual void OnWakeWordDetected(std::function<void(const std::string&)> callback) = 0;
    virtual void OnOutput(std::function<void(std::vector<int16_t>&&)> callback) = 0;
    virtual void OnVadStateChange(std::function<void(bool)> callback) = 0;
};
```

## Opus 编解码

### 编码配置

```cpp
#define AS_OPUS_ENC_CONFIG() {
    .sample_rate = ESP_AUDIO_SAMPLE_RATE_16K,
    .channel = ESP_AUDIO_MONO,
    .bits_per_sample = ESP_AUDIO_BIT16,
    .bitrate = ESP_OPUS_BITRATE_AUTO,
    .frame_duration = ESP_OPUS_ENC_FRAME_DURATION_60_MS,
    .application_mode = ESP_OPUS_ENC_APPLICATION_AUDIO,
    .complexity = 0,
    .enable_fec = false,
    .enable_dtx = true,
    .enable_vbr = true,
}
```

### 编解码任务

- 编码：PCM → Opus
- 解码：Opus → PCM
- 使用 `esp_opus_enc` 和 `esp_opus_dec` 库

## AEC（回声消除）

### AEC 模式

```cpp
enum AecMode {
    kAecOff,              // 关闭 AEC
    kAecOnDeviceSide,     // 设备端 AEC
    kAecOnServerSide,     // 服务器端 AEC
};
```

### 设备端 AEC

- 需要全双工音频编解码器
- `input_reference_ = true`（需要参考信号）
- 使用 AudioEngine 的 AEC 功能
- 仅 S3/P4 支持（AfeAudioEngine）

### 服务器端 AEC

- 需要时间戳同步
- 使用 `timestamp_queue_` 同步时间戳
- 服务器负责回声消除

## 唤醒词检测

### 唤醒词引擎

- 基于 ESP-SR 库
- 支持内置唤醒词和自定义唤醒词
- 唤醒词模型在 `srmodel_list_t` 中管理

### 检测流程

```
AudioEngine::Feed()
  → 内部唤醒词检测
  → 检测到唤醒词
  → OnWakeWordDetected 回调
  → AudioService 回调
  → Application::WakeWordInvoke()
```

### 自定义唤醒词

1. 使用 [xiaozhi-assets-generator](https://github.com/78/xiaozhi-assets-generator) 生成
2. 放入资源分区
3. 通过 `SetModelsList()` 加载

## VAD（语音活动检测）

### VAD 状态

```cpp
void OnVadStateChange(std::function<void(bool speaking)> callback);
```

### VAD 应用

- 自动停止监听（`kListeningModeAutoStop`）
- 检测说话开始和结束
- 减少无效音频传输

## 音频电源管理

### 电源定时器

```cpp
#define AUDIO_POWER_TIMEOUT_MS 15000
#define AUDIO_POWER_CHECK_INTERVAL_MS 1000
```

### 电源节省

- 15 秒无音频活动后进入低功耗模式
- 每秒检查一次活动状态
- 使用 `audio_power_timer_` 管理

## 关键规则

### 性能规则

1. **不要阻塞音频任务**
   - 音频任务是实时任务
   - 阻塞会导致音频断断续续

2. **避免大分配**
   - 音频路径中避免动态内存分配
   - 使用预分配缓冲区

3. **队列大小有上限**
   - 不要无界增长队列
   - 使用宏定义的上限

### 并发规则

1. **使用互斥锁保护共享状态**
   - `decoder_mutex_` 保护解码器
   - `audio_queue_mutex_` 保护队列

2. **使用条件变量同步**
   - `audio_queue_cv_` 用于队列同步
   - 避免忙等待

3. **原子操作**
   - `service_stopped_` 使用原子变量
   - `voice_detected_` 使用原子变量

### 采样率规则

1. **采集采样率**
   - 通常为 16kHz（唤醒词识别要求）
   - 不要随意改变

2. **播放采样率**
   - 由服务器配置（通常 24kHz）
   - 使用 `SetDecodeSampleRate()` 设置

3. **重采样**
   - 需要时使用 `input_resampler_` 和 `output_resampler_`
   - 避免不必要的重采样

## 调试和测试

### 调试统计

```cpp
struct DebugStatistics {
    uint32_t input_count = 0;
    uint32_t decode_count = 0;
    uint32_t encode_count = 0;
    uint32_t playback_count = 0;
    uint32_t encode_drop_count = 0;
};
```

### 音频测试模式

```cpp
#define AUDIO_TESTING_MAX_DURATION_MS 10000
```

- 用于设备端音频测试
- 限制测试时长为 10 秒

### 常见问题排查

1. **音频断断续续**
   - 检查是否阻塞音频任务
   - 检查队列是否溢出
   - 检查 CPU 使用率

2. **唤醒词检测失败**
   - 检查麦克风是否正常
   - 检查唤醒词模型是否加载
   - 检查环境噪音

3. **回声消除失败**
   - 检查是否全双工模式
   - 检查参考信号是否正常
   - 检查 AEC 模式配置

## 扩展音频系统

### 添加新编解码器

1. 继承 `AudioCodec`
2. 实现 `Read()` 和 `Write()`
3. 配置 I2S 和 DMA
4. 在开发板中返回实例

### 添加新音频引擎

1. 继承 `AudioEngine`
2. 实现所有纯虚函数
3. 在 `AudioService::InitializeAudioEngine()` 中选择

### 添加新音频效果

1. 在 `AudioEngine::Feed()` 中处理
2. 或使用 `AudioService` 的回调
3. 注意实时性要求
