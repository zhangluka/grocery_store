下面是一份**可直接指导开发实现的《Android AirPlay Receiver 详细设计说明书（DDS）》**。它面向你当前目标：**在 Android 5.1.1 电视（Cortex-A53 / Mali-450）上实现 iPhone AirPlay 镜像接收器**。协议栈复用 RPiPlay，视频解码使用 Android MediaCodec。

---

# Android AirPlay Receiver

## 详细设计说明书（DDS）

版本：1.0
目标设备：CultraHome Android TV
系统：Android 5.1.1

---

# 1 系统概述

本系统实现 **AirPlay Receiver**，用于接收 iPhone/iPad 投屏并在 Android TV 上播放。

核心能力：

* AirPlay 设备发现
* AirPlay 会话建立
* 视频 RTP 接收
* H264 硬件解码
* Surface 渲染
* 播放控制

---

# 2 系统架构

整体架构：

```
iPhone
 │
 │ AirPlay
 │
 ▼
RPiPlay
 │
 │ H264 NAL
 │
 ▼
JNI Bridge
 │
 ▼
MediaCodec
 │
 ▼
SurfaceView
 │
 ▼
Android Display
```

模块分层：

| 层级  | 模块          |
| --- | ----------- |
| 协议层 | AirPlay     |
| 接收层 | RPiPlay     |
| 解码层 | MediaCodec  |
| 渲染层 | SurfaceView |

---

# 3 AirPlay 协议设计

AirPlay 包含以下协议：

| 协议       | 作用    |
| -------- | ----- |
| mDNS     | 设备发现  |
| RTSP     | 会话控制  |
| FairPlay | 加密认证  |
| RTP      | 音视频传输 |

连接流程：

```
1 iPhone 发送 mDNS 查询
2 Receiver 返回 AirPlay 服务
3 RTSP 建立连接
4 FairPlay key exchange
5 RTP 视频流开始
```

---

# 4 视频处理设计

AirPlay 镜像视频参数：

| 参数  | 值         |
| --- | --------- |
| 编码  | H264      |
| 分辨率 | 1920×1080 |
| 帧率  | 30fps     |
| 码率  | 6–10 Mbps |

数据格式：

```
H264 NAL
```

NAL 示例：

```
00 00 00 01 SPS
00 00 00 01 PPS
00 00 00 01 IDR
```

视频处理流程：

```
RTP Packet
   │
NAL Parser
   │
JNI
   │
MediaCodec
   │
Surface
```

---

# 5 模块详细设计

## 5.1 AirplayServer

职责：

* 启动 AirPlay Receiver
* 初始化 RPiPlay
* 管理连接

接口：

```
start()
stop()
```

核心流程：

```
AirplayServer.start()
        │
        ▼
nativeStartServer()
        │
        ▼
RPiPlay start
```

---

## 5.2 JNI Bridge

作用：

```
C++ → Java 数据桥
```

数据结构：

```
H264 Frame
size
timestamp
```

JNI 方法：

```
onNativeVideoFrame(byte[] data, int size)
```

流程：

```
RPiPlay
 │
 ▼
send_frame()
 │
 ▼
JNI
 │
 ▼
Java
```

---

## 5.3 VideoDecoder

职责：

* H264 解码
* 输出到 Surface

初始化：

```
MediaCodec.createDecoderByType("video/avc")
```

配置：

```
resolution = 1920x1080
```

解码流程：

```
queueInputBuffer()
dequeueOutputBuffer()
releaseOutputBuffer()
```

---

## 5.4 Surface 渲染

渲染组件：

```
SurfaceView
```

优势：

* GPU 直通
* 无额外 copy
* 延迟低

渲染流程：

```
MediaCodec
   │
   ▼
Surface
   │
   ▼
Display
```

---

# 6 线程设计

系统线程：

```
Network Thread
RTP Parse Thread
Decode Thread
Render Thread
```

线程职责：

| 线程      | 作用         |
| ------- | ---------- |
| Network | 接收 RTP     |
| Parser  | H264 NAL   |
| Decoder | MediaCodec |
| Render  | Surface    |

---

# 7 网络设计

WiFi要求：

| 参数 | 值         |
| -- | --------- |
| 频段 | 5GHz      |
| 信号 | > -65 dBm |
| 带宽 | >20 Mbps  |

端口：

```
7000
7100
47000
```

RTP传输：

```
UDP
```

---

# 8 项目结构设计

工程结构：

```
android-airplay-receiver
│
├─ app
│
├─ java
│   ├─ MainActivity
│   ├─ AirplayServer
│   ├─ VideoDecoder
│   ├─ AudioDecoder
│   └─ AirplayCallback
│
├─ cpp
│   ├─ jni_bridge.cpp
│   ├─ video_pipeline.cpp
│   └─ rpiplay
│
└─ res
```

代码规模：

| 模块      | 行数   |
| ------- | ---- |
| Java    | 1200 |
| JNI     | 600  |
| RPiPlay | 1000 |

---

# 9 构建设计

Native 编译：

```
CMake
NDK
```

构建流程：

```
gradlew assembleDebug
```

生成：

```
app-debug.apk
```

---

# 10 安装部署

安装：

```
adb install app-debug.apk
```

启动：

```
adb shell am start
```

电视端运行：

```
AirPlay Receiver App
```

---

# 11 使用流程

iPhone：

```
控制中心
 → 屏幕镜像
 → CultraHome TV
```

连接成功：

```
iPhone screen → TV
```

---

# 12 性能指标

系统目标：

| 指标  | 目标         |
| --- | ---------- |
| 分辨率 | 1080p      |
| 帧率  | 30fps      |
| 码率  | 6–10 Mbps  |
| 延迟  | 150–250 ms |

CPU占用：

```
20%
```

---

# 13 关键优化

## RTP Buffer

默认：

```
500 ms
```

优化：

```
50 ms
```

---

## 硬件解码

必须使用：

```
MediaCodec
```

避免：

```
FFmpeg soft decode
```

---

## Surface 渲染

推荐：

```
SurfaceView
```

避免：

```
TextureView
```

---

# 14 风险与限制

设备限制：

| 项目  | 说明          |
| --- | ----------- |
| CPU | Cortex-A53  |
| GPU | Mali-450    |
| 系统  | Android 5.1 |

限制：

* 不支持 H265
* 1080p60 可能压力大

推荐：

```
1080p30
```

---

# 15 最终系统结构

完整系统：

```
iPhone
 │
AirPlay
 │
RPiPlay
 │
RTP
 │
H264 NAL
 │
JNI
 │
MediaCodec
 │
SurfaceView
 │
Android TV
```

---

✔ 该设计实现后，AirPlay 投屏效果将接近：

Apple TV 4K 的画质体验。

