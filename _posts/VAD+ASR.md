---
layout:       post
title:        "VAD + ASR：浏览器端实时语音识别方案解析"
subtitle:     "基于 WebAssembly + sherpa-onnx 的离线语音识别实现"
date:         2024-09-01 12:00:00
author:       "Chaoxu"
header-img:   "img/post-bg-2015.jpg"
header-mask:  0.3
catalog:      true
tags:
    - 技术
    - ASR
    - WebAssembly
    - AI
---

开箱即用
官方仓库

[https://github.com/k2-fsa/sherpa-onnx/releases](https://github.com/k2-fsa/sherpa-onnx/releases)
镜像地址
[https://sourceforge.net/projects/sherpa-onnx.mirror/files/v1.12.6/](https://sourceforge.net/projects/sherpa-onnx.mirror/files/v1.12.6/)

选的这个最小的中英文模型 sherpa-onnx-wasm-simd-1.12.6-vad-asr-zh_en-paraformer_small

### 核心技术栈

```
前端层：HTML5 + JavaScript
计算层：WebAssembly (C++编译)
AI模型：ONNX格式神经网络
音频处理：浏览器原生API
```

WebAssembly技术

定义：W3C标准的二进制指令格式，所有现代浏览器原生支持
特点：将C++高性能代码编译为浏览器可直接执行的字节码
优势：接近原生性能 + 浏览器安全沙盒 + 零安装使用

```
内网启动服务
npx http-server -p 6006 -a 0.0.0.0 -c-1 --cors

# 不执行，只是为了查看内网IP
ipconfig | findstr "IPv4"
http://10.58.92.84:6006/
```

``

# 代码流程解析

基于Web的实时语音识别应用，使用了WebAssembly和sherpa-onnx库。

## 1. **音频增强模块 (`enhanceAudio`函数)**

```javascript
function enhanceAudio(samples) {
  // 计算RMS(均方根)来评估音频强度
  const currentRMS = Math.sqrt(sum / samples.length)

  // 三种处理策略：
  if (currentRMS < 0.0015) {
    return null // 音频太弱，直接丢弃
  }
  if (currentRMS >= 0.12) {
    return samples // 音频足够强，不需要增强
  }

  // 需要增强：应用增益到目标RMS 0.15
  const gainFactor = targetRMS / currentRMS
  // 防止削峰，限制在[-1.0, 1.0]范围内
}
```

## 2. **音频处理管道流程**

```javascript
recorder.onaudioprocess = function(e) {
  // 1. 获取麦克风音频数据
  let samples = new Float32Array(e.inputBuffer.getChannelData(0))

  // 2. 重采样到16kHz
  samples = downsampleBuffer(samples, expectedSampleRate)

  // 3. 放入循环缓冲区
  buffer.push(samples)

  // 4. VAD检测 + 语音识别处理
  while (buffer.size() > vad.config.sileroVad.windowSize) {
    // VAD检测是否有语音
    vad.acceptWaveform(s)
    const isDetected = vad.isDetected()

    // 如果检测到语音结束，进行识别处理
    while (!vad.isEmpty()) {
      const segment = vad.front()
      // 音频增强
      const enhancedSamples = enhanceAudio(segment.samples)
      // 语音识别
      // 生成WAV文件
    }
  }
}
```

## 3. **VAD (Voice Activity Detection) 逻辑**

- 使用滑动窗口检测音频中的语音活动
- 当检测到语音时标记 `Speech detected`
- 当语音结束时，将积累的音频段送去识别
- 使用队列管理多个语音段

## 4. **离线语音识别器初始化**

```javascript
function initOfflineRecognizer() {
  // 支持多种模型格式：
  // - SenseVoice
  // - Whisper (encoder + decoder)
  // - Transducer
  // - Paraformer
  // - Moonshine
  // 等等...
}
```

## 5. **性能监控系统**

```javascript
let perfStats = {
  vadCallCount: 0,      // VAD调用次数
  vadTotalTime: 0,      // VAD总耗时
  asrCallCount: 0,      // ASR调用次数  
  asrTotalTime: 0,      // ASR总耗时
  enhanceCallCount: 0,  // 音频增强次数
  discardedCount: 0     // 丢弃的弱音频段数
}
```

## 6. **音频质量控制策略**

- **RMS阈值判断**: < 0.0015直接丢弃，>= 0.12不增强，中间范围进行增强
- **目标RMS**: 统一增强到0.15，确保识别质量
- **削峰保护**: 限制增益后的音频在[-1.0, 1.0]范围内
- **统计反馈**: 跟踪丢弃的音频段数量

## 7. **实时处理架构**

1. **音频捕获** → **重采样** → **缓冲**
2. **VAD检测** → **语音分段** → **音频增强**
3. **语音识别** → **结果显示** → **WAV生成**

这个系统的核心优势是：

- **智能音频增强**：解决麦克风收音不足问题
- **实时VAD**：准确分割语音段
- **性能监控**：实时跟踪各模块耗时
- **多模型支持**：兼容多种ONNX语音识别模型
- **质量控制**：自动丢弃质量过低的音频

# VAD ASR模块使用

在这段代码中，底层VAD和ASR组件的调用位置如下：

## **VAD (Voice Activity Detection) 调用**

### 1. **VAD初始化**

```javascript
Module.onRuntimeInitialized = function () {
  // 创建VAD实例 - 调用底层C++组件
  vad = createVad(Module)
  console.log('vad is created!', vad)
}
```

### 2. **VAD核心调用位置**

```javascript
recorder.onaudioprocess = function (e) {
  // ...音频预处理...

  while (buffer.size() > vad.config.sileroVad.windowSize) {
    const s = buffer.get(buffer.head(), vad.config.sileroVad.windowSize)

    // 🔥 关键调用1：向VAD输入音频数据
    vad.acceptWaveform(s)

    // 🔥 关键调用2：检测是否有语音活动
    const isDetected = vad.isDetected()

    buffer.pop(vad.config.sileroVad.windowSize)

    // 🔥 关键调用3：处理检测到的语音段
    while (!vad.isEmpty()) {
      const segment = vad.front()  // 获取语音段
      // ...处理逻辑...
      vad.pop()  // 移除已处理的语音段
    }
  }
}
```

### 3. **VAD重置调用**

```javascript
stopBtn.onclick = function () {
  vad.reset()  // 重置VAD状态
  // ...
}
```

## **ASR (Automatic Speech Recognition) 调用**

### 1. **ASR初始化**

```javascript
function initOfflineRecognizer() {
  let config = {
    modelConfig: {
      debug: 1,
      tokens: './tokens.txt'
    }
  }

  // 根据可用的模型文件选择配置
  if (fileExists('sense-voice.onnx') == 1) {
    config.modelConfig.senseVoice = {
      model: './sense-voice.onnx',
      useInverseTextNormalization: 1
    }
  }
  // ...其他模型配置...

  // 🔥 关键调用：创建离线识别器
  recognizer = new OfflineRecognizer(config, Module)
}
```

### 2. **ASR核心调用位置**

```javascript
while (!vad.isEmpty()) {
  const segment = vad.front()
  const enhancedSamples = enhanceAudio(segment.samples)

  if (enhancedSamples !== null) {
    // 🔥 关键调用1：创建识别流
    const stream = recognizer.createStream()

    // 🔥 关键调用2：向识别器输入音频数据
    stream.acceptWaveform(expectedSampleRate, enhancedSamples)

    // 🔥 关键调用3：执行语音识别
    recognizer.decode(stream)

    // 🔥 关键调用4：获取识别结果
    let recognitionResult = recognizer.getResult(stream)
    let text = recognitionResult.text

    // 🔥 关键调用5：释放流资源
    stream.free()
  }
}
```

## **底层组件架构**

这些调用实际上是通过 **WebAssembly (WASM)** 调用底层的C++实现：

```javascript
// WebAssembly模块加载和初始化
Module.onRuntimeInitialized = function () {
  // 底层组件都通过Module对象访问
  vad = createVad(Module)           // 调用C++的VAD实现
  recognizer = new OfflineRecognizer(config, Module)  // 调用C++的ASR实现
}
```

## **调用流程图**

```
音频输入 → VAD检测 → 语音分段 → 音频增强 → ASR识别 → 结果输出
    ↓         ↓         ↓         ↓         ↓         ↓
acceptWaveform() isDetected() front() enhanceAudio() decode() getResult()
```

## **关键特点**

1. **VAD**: 使用Silero VAD模型，通过滑动窗口持续检测语音活动
2. **ASR**: 支持多种ONNX模型 (Whisper, SenseVoice, Paraformer等)
3. **WebAssembly**: 所有底层计算都在WASM中执行，保证性能
4. **流式处理**: 实时处理音频流，无需等待完整录音

这种设计让整个系统能够在浏览器中实现高效的实时语音识别，底层的VAD和ASR都是经过优化的C++实现，通过WebAssembly桥接到JavaScript环境。
