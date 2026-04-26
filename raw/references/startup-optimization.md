# 首帧 / 起播速度优化

## 目录
1. [TTFF 拆解](#1-ttff-拆解)
2. [DNS 与连接优化](#2-dns-与连接优化)
3. [预加载策略](#3-预加载策略)
4. [预渲染（短视频核心）](#4-预渲染短视频核心)
5. [Android ExoPlayer 首帧优化](#5-android-exoplayer-首帧优化)
6. [iOS AVPlayer 首帧优化](#6-ios-avplayer-首帧优化)
7. [首帧图 / 封面图过渡](#7-首帧图--封面图过渡)
8. [服务端配合](#8-服务端配合)

---

## 1. TTFF 拆解

```
用户点击/滑入
    │
    ├── T1: DNS 解析 (0~200ms)
    ├── T2: TCP 握手 (RTT × 1, 约 20~100ms)
    ├── T3: TLS 握手 (RTT × 1~2, 约 40~200ms)
    ├── T4: HTTP 请求 + 响应首字节 (TTFB)
    ├── T5: 解析容器头（moov/索引）
    ├── T6: 解码首帧
    └── T7: 渲染上屏
    
TTFF = T1 + T2 + T3 + T4 + T5 + T6 + T7
目标：< 300ms（短视频）/ < 800ms（长视频）
```

**各阶段优化优先级（ROI 排序）**：
1. 预加载（消除 T1~T5）→ 收益最大
2. moov 前置（优化 T5）
3. HTTPDNS（优化 T1）
4. 连接复用（优化 T2/T3）
5. 硬解加速（优化 T6）

---

## 2. DNS 与连接优化

### HTTPDNS
```kotlin
// Android - 使用阿里云/腾讯 HTTPDNS SDK
val dnsManager = HttpDnsService.getService(context, accountId)
dnsManager.setPreResolveHosts(arrayListOf("video.example.com"))

// 拦截 OkHttp DNS 解析
val client = OkHttpClient.Builder()
    .dns { hostname ->
        val ipList = dnsManager.getIpsByHostAsync(hostname)
        if (ipList.isNotEmpty()) ipList.map { InetAddress.getByName(it) }
        else Dns.SYSTEM.lookup(hostname)
    }
    .build()
```

### 连接预热（Connection Pool）
```kotlin
// 在用户进入列表页时，提前建连 CDN
val preConnectUrls = listOf("https://cdn1.example.com", "https://cdn2.example.com")
preConnectUrls.forEach { url ->
    okHttpClient.newCall(Request.Builder().url(url).head().build()).enqueue(...)
}
```

### HTTP/2 & QUIC
- 启用 HTTP/2：单连接多路复用，避免 HoL blocking
- QUIC（HTTP/3）：弱网场景 0-RTT 重连，首帧提升 20~40%

---

## 3. 预加载策略

### 短视频预加载（仿抖音）

```kotlin
// 预加载管理器：维护一个滑动窗口
class PreloadManager(private val cacheDir: File) {
    private val preloadTasks = LinkedHashMap<String, PreloadTask>()
    private val maxPreloadCount = 3  // 预加载后 3 条
    private val preloadBytes = 512 * 1024  // 每条预加载 512KB（约够首帧）

    fun onListScroll(visibleIndex: Int, videoList: List<VideoItem>) {
        // 取消已不需要的预加载
        cancelOutOfWindowTasks(visibleIndex)
        
        // 启动新的预加载
        for (i in 1..maxPreloadCount) {
            val targetIndex = visibleIndex + i
            if (targetIndex < videoList.size) {
                preload(videoList[targetIndex].url)
            }
        }
    }

    private fun preload(url: String) {
        if (preloadTasks.containsKey(url)) return
        val task = PreloadTask(url, preloadBytes, cacheDir)
        preloadTasks[url] = task
        task.start()
    }
}
```

### 预加载量的权衡
| 预加载量 | 起播速度 | 流量消耗 | 推荐场景 |
|---|---|---|---|
| 512KB | 首帧加速 | 低 | WiFi + 4G 通用 |
| 2MB | 秒开 | 中 | WiFi 优先 |
| 全量缓存 | 离线播放 | 高 | 预下载功能 |

### 智能预加载（根据网络状态调整）
```kotlin
val preloadSize = when (networkType) {
    WIFI -> 2 * 1024 * 1024   // WiFi 预加载 2MB
    LTE  -> 512 * 1024         // 4G 预加载 512KB
    else -> 0                   // 3G/弱网不预加载
}
```

---

## 4. 预渲染（短视频核心）

> 预渲染 = 提前将视频数据解码并绘制到一个 **不可见的 Surface** 上，用户滑入时直接显示第一帧，无感切换。

### Android 实现

```kotlin
class VideoPreRenderer {
    private val surfacePool = SurfaceTexturePool(size = 3)

    // 用户还在看第 N 条时，提前渲染第 N+1 条
    fun preRender(videoUrl: String, onReady: (SurfaceTexture) -> Unit) {
        val surfaceTexture = surfacePool.acquire()
        val surface = Surface(surfaceTexture)
        
        val player = ExoPlayer.Builder(context).build().apply {
            setVideoSurface(surface)
            setMediaItem(MediaItem.fromUri(videoUrl))
            prepare()   // 此时就开始解码
        }
        
        player.addListener(object : Player.Listener {
            override fun onRenderedFirstFrame() {
                // 第一帧已渲染到 offscreen surface
                onReady(surfaceTexture)
            }
        })
    }

    // 用户滑入时：将 SurfaceTexture 绑定到可见的 TextureView
    fun attach(surfaceTexture: SurfaceTexture, textureView: TextureView) {
        textureView.surfaceTexture = surfaceTexture  // 直接切换，无黑屏
    }
}
```

### iOS 实现（AVPlayerItemOutput）
```swift
class VideoPreRenderer {
    private var prerenderedPlayers: [String: AVPlayer] = [:]
    
    func preRender(url: URL) {
        let item = AVPlayerItem(url: url)
        let player = AVPlayer(playerItem: item)
        player.automaticallyWaitsToMinimizeStalling = false
        // 预加载到 buffer
        player.preroll(atRate: 1.0) { [weak self] _ in
            // 首帧已在 buffer，等待上屏
        }
        prerenderedPlayers[url.absoluteString] = player
    }
    
    func player(for url: URL) -> AVPlayer? {
        return prerenderedPlayers[url.absoluteString]
    }
}
```

---

## 5. Android ExoPlayer 首帧优化

### moov 前置（MP4 关键）
```bash
# 服务端处理：将 moov box 移到文件头（必做！）
ffmpeg -i input.mp4 -movflags faststart -c copy output.mp4
```

> 未前置时，播放器需要下载完整文件才能找到 moov，TTFF 极差。

### ExoPlayer 配置优化
```kotlin
val loadControl = DefaultLoadControl.Builder()
    .setBufferDurationsMs(
        /* minBufferMs = */ 1_000,     // 最小缓冲 1s 即可开播（默认 50s，太保守）
        /* maxBufferMs = */ 30_000,
        /* bufferForPlaybackMs = */ 500,  // 积累 500ms 开始播放
        /* bufferForPlaybackAfterRebufferMs = */ 1_000  // 卡顿后 1s 恢复
    )
    .build()

val player = ExoPlayer.Builder(context)
    .setLoadControl(loadControl)
    .setRenderersFactory(
        DefaultRenderersFactory(context)
            .setExtensionRendererMode(EXTENSION_RENDERER_MODE_PREFER) // 优先硬解扩展
    )
    .build()
```

### 使用 MediaSource 复用（列表场景）
```kotlin
// 预创建 MediaSource，避免重复解析 URL
val mediaSourceFactory = DefaultMediaSourceFactory(cacheDataSourceFactory)
val preCreatedSource = mediaSourceFactory.createMediaSource(MediaItem.fromUri(url))

// 播放时直接使用，跳过 URL 解析耗时
player.setMediaSource(preCreatedSource)
player.prepare()  // 有缓存时几乎立即首帧
```

---

## 6. iOS AVPlayer 首帧优化

### 关键参数设置
```swift
let item = AVPlayerItem(url: url)

// 允许在网络不稳定时立即开始（不等待 stall-free 缓冲）
player.automaticallyWaitsToMinimizeStalling = false

// 设置缓冲策略（iOS 10+）
item.preferredForwardBufferDuration = 2.0  // 只预缓冲 2s，减少等待

// 优先解码关键帧（首帧加速）
let output = AVPlayerItemVideoOutput(pixelBufferAttributes: [
    kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA
])
item.add(output)
```

### AVAsset 预加载
```swift
// 提前加载 tracks 和 duration，避免播放时阻塞
let asset = AVURLAsset(url: url)
let keys = ["tracks", "duration", "playable"]
asset.loadValuesAsynchronously(forKeys: keys) {
    var error: NSError?
    let status = asset.statusOfValue(forKey: "tracks", error: &error)
    if status == .loaded {
        // 资源就绪，立即创建 PlayerItem
        let item = AVPlayerItem(asset: asset)
        DispatchQueue.main.async {
            self.player.replaceCurrentItem(with: item)
        }
    }
}
```

---

## 7. 首帧图 / 封面图过渡

> 在真正的首帧出现之前，用封面图填充，避免黑屏感知：

```kotlin
// Android：封面图 + 首帧回调隐藏
imageView.loadUrl(coverUrl)  // Glide 加载封面
player.addListener(object : Player.Listener {
    override fun onRenderedFirstFrame() {
        // 首帧已渲染，淡出封面图
        imageView.animate().alpha(0f).setDuration(150).start()
    }
})
```

```swift
// iOS
coverImageView.isHidden = false
NotificationCenter.default.addObserver(forName: .AVPlayerItemNewAccessLogEntry, 
    object: item, queue: .main) { _ in
    // 首帧出现，隐藏封面
    UIView.animate(withDuration: 0.15) {
        self.coverImageView.alpha = 0
    }
}
```

---

## 8. 服务端配合

| 服务端优化 | 效果 |
|---|---|
| moov faststart（MP4）| 减少 T5 50~200ms |
| 关键帧间隔 ≤ 2s | 首帧解码更快 |
| CDN 预热热门内容 | 减少回源，TTFB 降低 |
| HTTP/2 Server Push | 提前推送 moov 数据 |
| 小文件首包优化（≥128KB）| 首包即可解析头信息 |
| 封面图与视频同 CDN 域名 | 连接复用 |