# 播放器架构设计

## 目录
1. [整体分层架构](#1-整体分层架构)
2. [核心模块设计](#2-核心模块设计)
3. [线程模型](#3-线程模型)
4. [音视频同步（A/V Sync）](#4-音视频同步av-sync)
5. [状态机设计](#5-状态机设计)
6. [跨平台架构（FFmpeg 内核）](#6-跨平台架构ffmpeg-内核)

---

## 1. 整体分层架构

```
┌─────────────────────────────────────────┐
│              业务层 (Business Layer)      │
│  播放控制 / UI / 统计埋点 / 广告插播       │
├─────────────────────────────────────────┤
│              播放器 SDK 层               │
│  ┌──────────┐  ┌─────────┐  ┌────────┐ │
│  │  网络模块  │  │ 解封装  │  │ 解码器 │ │
│  │ HTTP/HLS  │  │Demuxer │  │Decoder │ │
│  └──────────┘  └─────────┘  └────────┘ │
├─────────────────────────────────────────┤
│              渲染层 (Render Layer)        │
│  视频渲染(OpenGL/Metal/Vulkan) + 音频输出 │
├─────────────────────────────────────────┤
│              OS / 硬件层                 │
│  MediaCodec / VideoToolbox / AudioTrack │
└─────────────────────────────────────────┘
```

### 关键设计原则
- **解耦**：网络、解码、渲染三大模块通过队列通信，互不阻塞
- **可替换**：解码器可在软解/硬解之间切换；渲染器支持 SurfaceView/TextureView 切换
- **可观测**：每层暴露 metrics 接口，便于 APM 采集

---

## 2. 核心模块设计

### 2.1 网络模块
```
NetworkStack
  ├── HttpEngine（OkHttp/NSURLSession/自研）
  ├── Cache（磁盘 + 内存两级）
  ├── CDN 调度（DNS/302/HTTPDNS）
  └── 带宽探测（用于 ABR 决策）
```

**关键参数**：
- `connectTimeout`：建连超时，推荐 3~5s
- `readTimeout`：读取超时，直播 < 2s，点播 < 10s
- TCP 初始窗口：与 CDN 协商放大（`SO_SNDBUF`）
- HTTP/2 多路复用：减少连接建立开销

### 2.2 解封装模块（Demuxer）
```
Demuxer
  ├── MP4/fMP4 Parser
  ├── MPEG-TS Parser（直播 HLS）
  ├── FLV Parser（RTMP）
  └── WebM/MKV Parser（DASH）
```

**fMP4 vs TS**：
- fMP4：随机访问更高效（moov box），适合点播 ABR 切换
- TS：实时性好，直播 HLS 常用

### 2.3 解码模块
```
DecoderManager
  ├── HardwareDecoder（MediaCodec / VideoToolbox）
  │   ├── 优先级最高，功耗低，4K/HDR 必选
  │   └── 注意：部分机型 H.265 硬解异常，需白名单
  ├── SoftwareDecoder（FFmpeg libavcodec）
  │   ├── 兼容性好，但功耗高
  │   └── 用于硬解 fallback
  └── DecodeStrategy
      ├── 超时自动切换软解
      └── 错误计数熔断
```

### 2.4 渲染模块

**Android**：
| 渲染方案 | 优点 | 缺点 |
|---|---|---|
| SurfaceView | 独立图层，性能最优 | 不支持动画/变换 |
| TextureView | 支持旋转/缩放/Alpha | 功耗略高，不能在 Window 上叠加 |
| SurfaceTexture | 灵活，可接 OpenGL | 需手动管理生命周期 |

**iOS**：
| 渲染方案 | 适用场景 |
|---|---|
| AVPlayerLayer | 系统播放，最省力 |
| Metal + CVPixelBuffer | 自定义滤镜、美颜 |
| OpenGLES（deprecated）| 旧代码维护 |

---

## 3. 线程模型

```
主线程（UI）
    │
    ├── 播放控制指令
    │
NetworkThread ──[packet queue]──▶ DemuxThread ──[frame queue]──▶ DecodeThread
                                                                      │
                                                              [decoded frame queue]
                                                                      │
                                                               RenderThread（VideoOutput）
                                                               AudioThread（AudioTrack/AVAudioEngine）
```

**关键队列设计**：
- `PacketQueue`：网络下载的原始 packet，容量控制网络缓存量
- `FrameQueue`：解码后的 YUV/RGBA 帧，容量控制内存占用
- 队列满时，网络/解码 **主动等待**，避免 OOM
- 队列空时，触发 buffering 事件，UI 显示 loading

**优先级设置**：
- AudioThread: `THREAD_PRIORITY_URGENT_AUDIO`（Android）/ `QOS_CLASS_USER_INTERACTIVE`（iOS）
- 视频渲染比音频优先级低一级，以音频为主时钟

---

## 4. 音视频同步（A/V Sync）

### 原理
以 **音频时钟** 为主时钟（人耳对音频不同步更敏感）：

```
AudioClock = 当前音频播放位置（PTS）

渲染视频帧时：
  diff = frame.pts - AudioClock
  
  if diff > +40ms:   // 视频超前，等待
      sleep(diff - 10ms)
  elif diff < -80ms: // 视频落后，丢帧追赶
      dropFrame()
  else:              // 正常范围，立即渲染
      render(frame)
```

### 直播场景特殊处理
- 直播无 PTS 修复参考，需要本地生成基准时钟
- 网络抖动缓冲（Jitter Buffer）：通常 1~3s，延迟与流畅的权衡
- 追帧策略：积压超过阈值时，静音快速播放追赶

### 常见问题
| 现象 | 原因 | 解决 |
|---|---|---|
| 嘴型对不上 | PTS 不准 / 音频解码延迟大 | 校正 PTS，减小音频 buffer |
| 视频跳帧 | 解码耗时超过帧间隔 | 降低分辨率 / 启用硬解 |
| 音频断断续续 | AudioTrack underrun | 增大音频 buffer，提升 AudioThread 优先级 |

---

## 5. 状态机设计

```
IDLE ──prepare()──▶ PREPARING ──prepared()──▶ PREPARED
                                                  │
                                              play() / auto
                                                  ▼
                                              PLAYING ◀──────────────────┐
                                                  │                       │
                                            buffer empty             resume()
                                                  ▼                       │
                                            BUFFERING ──buffer full──▶ PLAYING
                                                  │
                                              pause()
                                                  ▼
                                              PAUSED ──play()──▶ PLAYING
                                                  │
                                             stop() / error
                                                  ▼
                                              STOPPED / ERROR
```

**事件回调设计**：
```kotlin
interface PlayerEventListener {
    fun onStateChanged(state: PlayerState)
    fun onBufferingUpdate(percent: Int)
    fun onFirstFrameRendered(ttffMs: Long)  // 首帧回调，用于埋点
    fun onError(error: PlayError)
    fun onCompletion()
    fun onSeekComplete()
}
```

---

## 6. 跨平台架构（FFmpeg 内核）

适用于 IJKPlayer / 自研跨端播放器：

```
┌─────────────┐     ┌─────────────┐
│  Android JNI│     │  iOS ObjC   │
│  Java/Kotlin│     │  Swift      │
└──────┬──────┘     └──────┬──────┘
       │                   │
       └─────────┬─────────┘
                 ▼
         C/C++ Core Layer
         ┌─────────────┐
         │  FFmpeg      │  解封装 + 软解
         │  libavformat │
         │  libavcodec  │
         └──────┬───────┘
                │
         ┌──────▼───────┐
         │  渲染适配层   │
         │  ANativeWindow│(Android)
         │  CVPixelBuffer│(iOS)
         └──────────────┘
```

**跨平台权衡**：
- 核心解码逻辑共用 C 层，减少重复
- 渲染、DRM、音频输出必须平台原生实现
- 建议仅在需要高度定制（滤镜、协议扩展）时选择跨端内核