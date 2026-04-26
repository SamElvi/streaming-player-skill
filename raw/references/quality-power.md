# 画质增强、功耗与内存优化

## 画质优化

### HDR 播放（iOS）
```swift
// 检测 HDR 支持
if AVPlayer.eligibleForHDRPlayback {
    // 设备支持 HDR，优先选择 HDR 流
    item.preferredPeakBitRate = 0  // 不限速，让系统选最高质量
}
```

### 画质增强（超分）
- Android：MediaCodec 输出后接 OpenGL 着色器做超分
- iOS：Core Image + Metal Performance Shaders
- 建议：仅在充电 + WiFi + 旗舰机型开启，避免发热

### 色彩管理
```kotlin
// Android 10+ 色域适配
if (Build.VERSION.SDK_INT >= 29) {
    surface.setFrameRate(60f, Surface.FRAME_RATE_COMPATIBILITY_DEFAULT)
}
```

---

## 功耗优化

### 优先使用硬件解码
```kotlin
// ExoPlayer：强制硬解，软解 fallback
DefaultRenderersFactory(context)
    .setExtensionRendererMode(EXTENSION_RENDERER_MODE_PREFER)
    // 硬解失败时自动切软解
```

### 帧率控制
```kotlin
// 非全屏/后台时降低帧率
fun onAppBackground() {
    player.setVideoFrameMetadataListener { ... }
    // 或 pause 视频，仅保持音频
}
```

### 后台播放（仅音频）
```kotlin
// Android：进入后台时关闭视频渲染，节省 GPU
player.setVideoSurface(null)   // 断开 surface
// 音频继续，通过 WakeLock 保持 CPU

// iOS：
AVAudioSession.sharedInstance().setCategory(.playback, mode: .default)
// Background Modes → Audio, AirPlay, and Picture in Picture
```

---

## 内存优化

### 解码帧队列控制
```kotlin
// 限制 decoded frame queue 深度，避免 OOM
val maxQueuedFrames = when {
    resolution.height >= 2160 -> 3   // 4K：最多 3 帧
    resolution.height >= 1080 -> 5   // 1080p：5 帧
    else                      -> 8
}
```

### Surface 生命周期管理
```kotlin
// Activity onStop 时释放 Surface
override fun onStop() {
    super.onStop()
    player.setVideoSurface(null)  // 释放 Surface，但 player 继续缓冲
}

override fun onStart() {
    super.onStart()
    player.setVideoSurface(surfaceView.holder.surface)
}
```

### 多 Player 实例内存
```
每个 ExoPlayer 实例约占：
  - 解码器缓冲：10~50MB（分辨率相关）
  - 网络缓冲：视 maxBufferMs 而定
  
建议：短视频播放器池最多 3~5 个实例
```

---

## 性能监控

```kotlin
// 关键帧率指标
player.addAnalyticsListener(object : AnalyticsListener {
    override fun onDroppedVideoFrames(eventTime: EventTime, count: Int, elapsed: Long) {
        // 丢帧告警：1s 内丢帧 > 2 帧时上报
        if (count > 2) {
            metrics.record("dropped_frames", count)
        }
    }
})
```