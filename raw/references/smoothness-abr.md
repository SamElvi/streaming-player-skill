# 卡顿治理 & ABR 自适应码率

## 目录
1. [卡顿分类与根因](#1-卡顿分类与根因)
2. [缓冲策略设计](#2-缓冲策略设计)
3. [ABR 算法详解](#3-abr-算法详解)
4. [Android ExoPlayer ABR 配置](#4-android-exoplayer-abr-配置)
5. [iOS AVPlayer 自适应流](#5-ios-avplayer-自适应流)
6. [卡顿监控与告警](#6-卡顿监控与告警)
7. [直播场景追帧策略](#7-直播场景追帧策略)

---

## 1. 卡顿分类与根因

```
卡顿（Buffering / Stall）
    ├── 网络带宽不足
    │   ├── 码率选择过高（ABR 决策滞后）
    │   └── CDN 节点带宽打满（切换节点）
    ├── 解码性能不足
    │   ├── 高分辨率软解 CPU 跑满
    │   ├── 硬解器队列积压
    │   └── 主线程 GC / ANR 干扰解码线程
    ├── 服务端推流问题（直播）
    │   ├── 推流端网络波动
    │   └── GOP 过长（关键帧间隔 > 5s）
    └── 内存压力
        └── OOM 触发 GC，渲染帧丢失
```

**定位步骤**：
1. 抓 `BufferingStarted` 时的 **网络带宽**（是否 < 当前码率 ×1.5）
2. 检查 **DecodeQueue 深度**（是否为 0，解码跟不上）
3. 排查 **系统 CPU/内存**（是否有其他进程竞争）

---

## 2. 缓冲策略设计

### 三段式缓冲（推荐）

```
[空]──────[低水位]────────[高水位]────────[满]
 0%         20%              80%          100%

空 → 低水位：  BUFFERING 状态，显示 loading，全力下载
低→ 高水位：  PLAYING 状态，后台继续填充
高水位 → 满：  限速下载，避免内存膨胀
```

```kotlin
// ExoPlayer DefaultLoadControl 配置
DefaultLoadControl.Builder()
    .setBufferDurationsMs(
        minBufferMs = 2_000,   // 低水位：积累 2s 即可播
        maxBufferMs = 60_000,  // 高水位：最多缓 60s（WiFi）
        bufferForPlaybackMs = 1_000,
        bufferForPlaybackAfterRebufferMs = 2_000
    )
    // 网络差时动态降低 maxBuffer，避免浪费流量
    .setTargetBufferBytes(C.LENGTH_UNSET)
    .build()
```

### 网络感知动态调整
```kotlin
val maxBufferMs = when {
    networkBandwidth > 5_000_000 -> 60_000  // 5Mbps+ → 缓 60s
    networkBandwidth > 1_000_000 -> 30_000  // 1Mbps+ → 缓 30s
    else                         -> 10_000  // 弱网 → 缓 10s，保守
}
```

---

## 3. ABR 算法详解

### 3.1 基于吞吐量的 ABR（Throughput-based）

```
每次 segment 下载完成后更新带宽估计：
  bandwidth_estimate = segment_size / download_time × 平滑系数

下一个 segment 选择满足：
  bitrate[i] ≤ bandwidth_estimate × safety_factor（通常 0.8）
```

**优点**：实现简单，响应快
**缺点**：带宽估计有滞后，切换频繁

### 3.2 基于 Buffer 的 ABR（Buffer-based, BBA）

```
buffer_level = 当前缓冲时长

if buffer_level < 10s:    → 选最低码率，优先保流畅
elif buffer_level < 20s:  → 维持当前码率
elif buffer_level > 40s:  → 尝试升码率
```

**优点**：切换稳定，不依赖带宽估计精度
**缺点**：升码率慢，画质爬升延迟

### 3.3 BOLA（Buffer Occupancy based Lyapunov Algorithm）

Netflix 等公司使用，兼顾 QoE（用户体验质量）：
- 最大化视频质量 + 最小化重缓冲
- 基于控制论，理论上最优

### 3.4 混合 ABR（实际推荐）

```python
def select_bitrate(buffer_level, bandwidth, current_bitrate, available_bitrates):
    # 1. 带宽上限
    max_by_bw = bandwidth * 0.8
    
    # 2. Buffer 安全下限
    if buffer_level < 5:
        return available_bitrates[0]  # 最低，保命
    
    # 3. 综合选择
    candidates = [r for r in available_bitrates if r <= max_by_bw]
    
    # 4. 防抖：切换需要连续 3 次满足条件（避免反复横跳）
    if should_upgrade(candidates, current_bitrate, consecutive_count):
        return candidates[-1]
    return current_bitrate
```

---

## 4. Android ExoPlayer ABR 配置

### 自定义带宽估计
```kotlin
val bandwidthMeter = DefaultBandwidthMeter.Builder(context)
    .setInitialBitrateEstimate(C.NETWORK_TYPE_WIFI, 5_000_000L)  // WiFi 初始估计 5Mbps
    .setInitialBitrateEstimate(C.NETWORK_TYPE_4G, 2_000_000L)
    .build()
```

### 自定义 TrackSelector（码率选择）
```kotlin
val trackSelector = DefaultTrackSelector(context).apply {
    parameters = buildUponParameters()
        .setMaxVideoBitrate(maxBitrate)   // 强制上限（省流模式）
        .setMinVideoBitrate(minBitrate)
        .setForceHighestSupportedBitrate(false)  // 不强制最高，由 ABR 决定
        .setViewportSizeToPhysicalDisplaySize(activity, true)  // 根据屏幕分辨率过滤
        .build()
}
```

### 监听 ABR 切换事件（用于埋点）
```kotlin
player.addAnalyticsListener(object : AnalyticsListener {
    override fun onVideoInputFormatChanged(
        eventTime: EventTime,
        format: Format,
        decoderReuseEvaluation: DecoderReuseEvaluation?
    ) {
        analyticsLogger.log("ABR_SWITCH", mapOf(
            "bitrate" to format.bitrate,
            "resolution" to "${format.width}x${format.height}",
            "buffer_level" to player.totalBufferedDuration
        ))
    }
})
```

---

## 5. iOS AVPlayer 自适应流

### HLS 质量选择配置
```swift
// 限制最高分辨率（省流模式）
player.currentItem?.preferredPeakBitRate = 2_000_000  // 限速 2Mbps

// 允许蜂窝网络播放（默认只 WiFi 播高码率）
player.currentItem?.preferredMaximumResolutionForExpensiveNetworks = CGSize(width: 1280, height: 720)

// 关闭自动等待（减少首帧等待）
player.automaticallyWaitsToMinimizeStalling = false
```

### 监听码率切换
```swift
// 观察 accessLog 获取 ABR 事件
NotificationCenter.default.addObserver(
    forName: .AVPlayerItemNewAccessLogEntry,
    object: playerItem,
    queue: .main
) { [weak self] _ in
    if let event = self?.playerItem.accessLog()?.events.last {
        print("Current bitrate: \(event.indicatedBitrate)")
        print("Switch count: \(event.numberOfServerAddressChanges)")
    }
}
```

---

## 6. 卡顿监控与告警

### 核心指标
```kotlin
data class PlaybackMetrics(
    val bufferingCount: Int,        // 卡顿次数
    val bufferingDurationMs: Long,  // 总卡顿时长
    val playDurationMs: Long,       // 总播放时长
) {
    // 卡顿率 = 总卡顿时长 / (总播放时长 + 总卡顿时长)
    val bufferingRatio: Float
        get() = bufferingDurationMs.toFloat() / (playDurationMs + bufferingDurationMs)
}
```

### 埋点时机
| 事件 | 埋点字段 |
|---|---|
| 卡顿开始 | timestamp, buffer_level, bandwidth, bitrate |
| 卡顿结束 | duration, recovery_bandwidth |
| ABR 切换 | old_bitrate, new_bitrate, reason |
| 播放失败 | error_code, url, cdn_ip |

### 告警阈值
- 卡顿率 > 1%：P2 告警
- 卡顿率 > 3%：P1 告警，自动切 CDN
- TTFF > 3s：P2 告警

---

## 7. 直播场景追帧策略

直播延迟积压时，需要追帧：

```kotlin
class LiveLatencyController(private val player: ExoPlayer) {
    private val targetLatencyMs = 3_000L  // 目标延迟 3s
    private val maxLatencyMs = 8_000L     // 超过 8s 触发追帧
    
    fun onPlaybackStats() {
        val currentLatency = calculateCurrentLatency()
        
        when {
            currentLatency > maxLatencyMs -> {
                // 严重积压：直接 seek 到最新位置
                player.seekToDefaultPosition()
            }
            currentLatency > targetLatencyMs + 2_000 -> {
                // 轻微积压：加速播放追赶（无声音变调）
                player.setPlaybackSpeed(1.25f)
            }
            currentLatency < targetLatencyMs - 1_000 -> {
                // 延迟过低（超前）：减速
                player.setPlaybackSpeed(0.9f)
            }
            else -> {
                player.setPlaybackSpeed(1.0f)
            }
        }
    }
}
```