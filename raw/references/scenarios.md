# 典型场景方案

## 目录
1. [短视频场景（抖音/快手/Reels）](#1-短视频场景)
2. [直播场景](#2-直播场景)
3. [长视频场景（爱奇艺/Netflix）](#3-长视频场景)
4. [场景对比速查表](#4-场景对比速查表)

---

## 1. 短视频场景

### 核心体验目标
- **秒开率 > 95%**（用户滑入后 300ms 内首帧）
- **滑动流畅**：60fps 无卡顿
- **零黑屏切换**

### 架构方案：播放器池 + 预加载 + 预渲染

```
VideoPool（3~5个 ExoPlayer/AVPlayer 实例循环复用）
    ├── 当前播放：Player[0] → 绑定可见 Surface
    ├── 预渲染：Player[1] → 已解码第一帧（offscreen）
    └── 预加载：Player[2] → 正在下载数据到 Cache
```

### Android 实现要点

```kotlin
class ShortVideoPlayerPool(context: Context, poolSize: Int = 3) {
    private val players = ArrayDeque<ExoPlayer>()
    
    init {
        repeat(poolSize) {
            players.add(buildPlayer(context))
        }
    }
    
    private fun buildPlayer(context: Context) = ExoPlayer.Builder(context)
        .setLoadControl(
            DefaultLoadControl.Builder()
                .setBufferDurationsMs(500, 10_000, 250, 500)  // 激进的低缓冲，快速开播
                .build()
        )
        .build()
    
    // 滑动时：当前 player 暂停，下一个 player（已预渲染）接管
    fun swap(
        currentHolder: RecyclerView.ViewHolder,
        nextHolder: RecyclerView.ViewHolder
    ) {
        val currentPlayer = players.removeFirst()
        currentPlayer.pause()
        players.addLast(currentPlayer)  // 放回池尾
        
        val nextPlayer = players.first() // 已预渲染的 player
        nextPlayer.play()
        bindToView(nextPlayer, nextHolder)
    }
}
```

### SurfaceTexture 复用（关键：避免黑帧）
```kotlin
// 使用 SurfaceTexturePool，避免 TextureView 重建导致的黑帧
class SurfaceTexturePool(private val size: Int) {
    private val pool = ArrayDeque<SurfaceTexture>()
    
    fun acquire(): SurfaceTexture = pool.removeFirstOrNull() ?: SurfaceTexture(0)
    
    fun release(st: SurfaceTexture) {
        if (pool.size < size) pool.addLast(st)
        else st.release()
    }
}
```

### RecyclerView 滚动监听
```kotlin
recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
    override fun onScrollStateChanged(rv: RecyclerView, newState: Int) {
        if (newState == RecyclerView.SCROLL_STATE_IDLE) {
            val centerItem = findCenterVisibleItemPosition()
            playerManager.playAt(centerItem)
            preloadManager.updateWindow(centerItem)  // 更新预加载窗口
        }
    }
})
```

### iOS 短视频方案
```swift
class ShortVideoViewController: UIPageViewController {
    private var playerCache: [Int: AVPlayer] = [:]
    
    override func setViewControllers(_ vcs: [UIViewController],
                                      direction: NavigationDirection,
                                      animated: Bool,
                                      completion: ((Bool) -> Void)? = nil) {
        // 切换前，预加载下一个
        preloadNext(currentIndex + 1)
        super.setViewControllers(vcs, direction: direction, animated: animated)
    }
    
    private func preloadNext(_ index: Int) {
        guard index < videos.count,
              playerCache[index] == nil else { return }
        let player = AVPlayer(url: videos[index].url)
        player.automaticallyWaitsToMinimizeStalling = false
        player.preroll(atRate: 1.0, completionHandler: nil)
        playerCache[index] = player
    }
}
```

---

## 2. 直播场景

### 核心体验目标
- **延迟 < 3s**（标准直播）/ **< 1s**（互动直播）
- **卡顿率 < 0.3%**
- **首帧 < 1s**

### 协议选型

| 协议 | 延迟 | 适用场景 |
|---|---|---|
| RTMP | 1~3s | 主播推流（服务端接收）|
| HLS（标准）| 10~30s | 普通直播观看（高兼容）|
| LL-HLS | 1~3s | 低延迟直播（iOS 原生支持）|
| RTMP over HTTP-FLV | 1~3s | Android 直播观看（主流）|
| WebRTC | < 200ms | 连麦、超低延迟互动 |

### HTTP-FLV 直播播放（Android）

```kotlin
// ExoPlayer + FLV 扩展（或使用 IjkPlayer）
val dataSourceFactory = DefaultHttpDataSource.Factory()
    .setConnectTimeoutMs(3_000)
    .setReadTimeoutMs(5_000)
    .setAllowCrossProtocolRedirects(true)

val mediaItem = MediaItem.Builder()
    .setUri("http://live.example.com/live/stream.flv")
    .build()

// FLV 需要自定义 MediaSource（ExoPlayer 内置不支持 FLV，需扩展）
val mediaSource = ProgressiveMediaSource.Factory(dataSourceFactory)
    .createMediaSource(mediaItem)
```

### LL-HLS 低延迟直播（iOS）

```swift
// iOS 17+ AVPlayer 原生支持 LL-HLS
let item = AVPlayerItem(url: URL(string: "https://live.example.com/hls/stream.m3u8")!)

// 关闭自动等待，允许在 stall 边缘播放
player.automaticallyWaitsToMinimizeStalling = false

// LL-HLS 关键：服务端分片时长 ≤ 2s，部分推送（Partial Segments）
```

### 直播断线重连
```kotlin
class LiveReconnectStrategy {
    private var retryCount = 0
    private val maxRetries = 5
    
    fun onPlaybackError(error: PlaybackException) {
        if (retryCount >= maxRetries) {
            notifyUserError()
            return
        }
        
        val delayMs = when (retryCount) {
            0 -> 500L    // 首次：500ms 快速重试
            1 -> 1_000L
            2 -> 2_000L
            else -> 5_000L  // 后续：指数退避
        }
        
        Handler(Looper.getMainLooper()).postDelayed({
            player.prepare()  // 重新准备（自动重连）
            retryCount++
        }, delayMs)
    }
    
    fun onPlaybackStarted() {
        retryCount = 0  // 播放成功，重置计数
    }
}
```

### 直播 GOP 缓存优化（首帧加速）
> 服务端对每个直播流缓存最近一个 GOP（2s），新观众连接时直接推送，无需等待下一个关键帧。

---

## 3. 长视频场景

### 核心体验目标
- **高画质**：4K/HDR 支持
- **省流**：智能 ABR，不浪费流量
- **续播准确**：多端进度同步
- **Seek 快**：任意位置跳转 < 1s

### DASH 点播配置

```kotlin
// DASH 多码率流
val dashUri = Uri.parse("https://cdn.example.com/video/manifest.mpd")
val mediaSource = DashMediaSource.Factory(dataSourceFactory)
    .createMediaSource(MediaItem.fromUri(dashUri))
```

### 分段缓存（大文件不全量缓存）
```kotlin
val cacheDataSourceFactory = CacheDataSource.Factory()
    .setCache(SimpleCache(
        File(context.cacheDir, "video_cache"),
        LeastRecentlyUsedCacheEvictor(500 * 1024 * 1024L)  // 500MB 上限
    ))
    .setFlags(CacheDataSource.FLAG_IGNORE_CACHE_ON_ERROR)
    .setUpstreamDataSourceFactory(httpDataSourceFactory)
```

### Seek 优化
```kotlin
// ExoPlayer seek 模式：精确 vs 快速
// 快速 seek（跳到最近关键帧，有 1~2s 误差，但响应快）
player.seekTo(positionMs)  // 默认快速

// 精确 seek（跳到精确时间，解码耗时较长）
player.setSeekParameters(SeekParameters.EXACT)
player.seekTo(positionMs)
```

### 多清晰度选择 UI
```kotlin
// 获取可用清晰度列表
val tracks = player.currentTracks
val videoTrackGroups = tracks.groups.filter { it.type == C.TRACK_TYPE_VIDEO }

val resolutions = videoTrackGroups.flatMap { group ->
    (0 until group.length).map { i ->
        group.getTrackFormat(i).let { fmt ->
            Resolution(width = fmt.width, height = fmt.height, bitrate = fmt.bitrate)
        }
    }
}.distinctBy { it.height }.sortedByDescending { it.height }

// 用户选择后覆盖 ABR
fun setQuality(height: Int) {
    val params = trackSelector.parameters.buildUpon()
        .setMaxVideoSize(Int.MAX_VALUE, height)
        .setMinVideoSize(0, height)
        .build()
    trackSelector.parameters = params
}
```

### 续播与进度同步
```kotlin
// 播放进度上报（每 10s 上报一次，退出时立即上报）
class ProgressReporter(private val player: ExoPlayer) {
    private val handler = Handler(Looper.getMainLooper())
    private val reportRunnable = object : Runnable {
        override fun run() {
            if (player.isPlaying) {
                api.reportProgress(videoId, player.currentPosition)
                handler.postDelayed(this, 10_000L)
            }
        }
    }
    fun start() = handler.post(reportRunnable)
    fun stop() {
        handler.removeCallbacks(reportRunnable)
        api.reportProgress(videoId, player.currentPosition)  // 最终上报
    }
}
```

---

## 4. 场景对比速查表

| 维度 | 短视频 | 直播 | 长视频 |
|---|---|---|---|
| 首帧目标 | < 300ms | < 1s | < 1s |
| 卡顿容忍 | 极低（滑走） | 低（影响体验）| 中（seek 可接受）|
| 缓冲策略 | 极小缓冲（500ms）| 1~3s 延迟缓冲 | 大缓冲（30~60s）|
| ABR | 固定码率为主 | 带宽自适应 | DASH ABR |
| 预加载 | 必须（前2~3条）| 无（实时）| 有（分段预加载）|
| 协议 | HLS/MP4 | RTMP/FLV/LL-HLS | HLS/DASH |
| DRM | 可选 | 少见 | 必须（版权内容）|
| 多清晰度 | 少见 | 有（超清/标清）| 核心功能 |