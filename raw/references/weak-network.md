# 弱网对抗与断网续播

## 目录
1. [弱网检测](#1-弱网检测)
2. [弱网播放策略](#2-弱网播放策略)
3. [断网续播](#3-断网续播)
4. [QUIC / HTTP3 对抗弱网](#4-quic--http3-对抗弱网)
5. [网络切换处理（WiFi → 4G）](#5-网络切换处理)

---

## 1. 弱网检测

```kotlin
// Android 实时带宽监控
class NetworkQualityMonitor(private val bandwidthMeter: BandwidthMeter) {
    
    enum class NetworkQuality { EXCELLENT, GOOD, POOR, OFFLINE }
    
    fun getCurrentQuality(): NetworkQuality {
        val bps = bandwidthMeter.bitrateEstimate
        return when {
            bps <= 0            -> NetworkQuality.OFFLINE
            bps < 500_000       -> NetworkQuality.POOR      // < 500Kbps
            bps < 2_000_000     -> NetworkQuality.GOOD      // < 2Mbps
            else                -> NetworkQuality.EXCELLENT
        }
    }
}
```

```swift
// iOS NWPathMonitor（iOS 12+）
let monitor = NWPathMonitor()
monitor.pathUpdateHandler = { path in
    if path.status == .unsatisfied {
        self.handleOffline()
    } else if path.isExpensive {
        // 使用蜂窝网络，降低码率
        self.setMaxBitrate(1_000_000)
    }
}
monitor.start(queue: DispatchQueue.global())
```

---

## 2. 弱网播放策略

### 自动降画质
```kotlin
fun applyWeakNetworkStrategy(quality: NetworkQuality) {
    val params = trackSelector.parameters.buildUpon()
    when (quality) {
        POOR -> {
            params.setMaxVideoSize(854, 480)   // 强制 480p
                  .setMaxVideoBitrate(800_000)  // 限速 800Kbps
        }
        OFFLINE -> {
            // 尝试离线缓存播放
            switchToOfflineCache()
        }
        else -> {
            params.clearVideoSizeConstraints()  // 恢复正常
        }
    }
    trackSelector.parameters = params.build()
}
```

### 预缓冲量动态调整
```kotlin
// 弱网时增大缓冲，以时间换流畅
val bufferMs = when (quality) {
    POOR      -> 10_000   // 10s 缓冲，避免频繁卡顿
    GOOD      -> 30_000
    EXCELLENT -> 60_000
}
```

### HTTP 重试与多 CDN 切换
```kotlin
class RobustDataSource(private val cdnList: List<String>) : DataSource {
    private var currentCdnIndex = 0
    private var retryCount = 0
    
    override fun read(buffer: ByteArray, offset: Int, length: Int): Int {
        return try {
            upstream.read(buffer, offset, length)
        } catch (e: IOException) {
            if (retryCount < 3) {
                retryCount++
                reconnect()  // 同 CDN 重试
                read(buffer, offset, length)
            } else {
                // 切换到下一个 CDN
                currentCdnIndex = (currentCdnIndex + 1) % cdnList.size
                retryCount = 0
                reconnect(cdnList[currentCdnIndex])
                read(buffer, offset, length)
            }
        }
    }
}
```

---

## 3. 断网续播

### 本地缓存续播
```kotlin
// 离线缓存下载（用户主动缓存）
class OfflineDownloadManager(context: Context) {
    private val downloadManager = DownloadManager(
        context,
        DefaultDownloadIndex(StandaloneDatabaseProvider(context)),
        DefaultDownloaderFactory(cacheDataSourceFactory, Executor { it.run() })
    )
    
    fun downloadVideo(mediaItem: MediaItem) {
        val request = DownloadRequest.Builder(mediaItem.mediaId, mediaItem.localConfiguration!!.uri)
            .build()
        DownloadService.sendAddDownload(context, MyDownloadService::class.java, request, false)
    }
    
    // 播放时优先读取本地缓存
    fun buildPlayerWithOfflineSupport(): ExoPlayer {
        val dataSourceFactory = CacheDataSource.Factory()
            .setCache(downloadCache)
            .setUpstreamDataSourceFactory(httpDataSourceFactory)
            .setCacheWriteDataSinkFactory(null)  // 只读缓存，不写
        // ...
    }
}
```

### 断点续传（Range 请求）
```kotlin
// HTTP Range 请求，从断点处恢复下载
val request = Request.Builder()
    .url(videoUrl)
    .header("Range", "bytes=${downloadedBytes}-")  // 从已下载位置继续
    .build()
```

### iOS 后台下载
```swift
// 使用 URLSessionDownloadTask + background configuration
let config = URLSessionConfiguration.background(withIdentifier: "video.download")
config.isDiscretionary = false  // 不延迟，立即下载
let session = URLSession(configuration: config, delegate: self, delegateQueue: nil)
let task = session.downloadTask(with: url)
task.resume()

// 应用被 kill 后，系统会在下载完成时唤醒应用
func application(_ application: UIApplication,
                 handleEventsForBackgroundURLSession identifier: String,
                 completionHandler: @escaping () -> Void) {
    backgroundCompletionHandler = completionHandler
}
```

---

## 4. QUIC / HTTP3 对抗弱网

**QUIC 优势**：
- 0-RTT 重连：网络切换时无需重新握手
- 无 HoL 阻塞：多路复用在 UDP 层，丢包不阻塞其他流
- 弱网首帧提升 15~30%

```kotlin
// OkHttp 启用 QUIC（需配合服务端支持）
val client = OkHttpClient.Builder()
    .protocols(listOf(Protocol.QUIC, Protocol.HTTP_2, Protocol.HTTP_1_1))
    .build()
```

---

## 5. 网络切换处理

```kotlin
// WiFi ↔ 4G 切换时无缝续播
class NetworkSwitchHandler(private val player: ExoPlayer) {
    
    fun onNetworkChanged(newType: NetworkType) {
        val currentPosition = player.currentPosition
        val wasPlaying = player.isPlaying
        
        // 保存当前播放位置
        savedPosition = currentPosition
        
        // 重新准备（可能需要切换 CDN 或重连）
        player.prepare()
        player.seekTo(currentPosition)
        if (wasPlaying) player.play()
        
        // 根据新网络类型调整策略
        applyWeakNetworkStrategy(qualityFor(newType))
    }
}
```

---

# DRM 内容保护

## 目录
1. [DRM 方案选型](#1-drm-方案选型)
2. [Android Widevine](#2-android-widevine)
3. [iOS FairPlay](#3-ios-fairplay)
4. [离线 DRM（下载播放）](#4-离线-drm)

---

## 1. DRM 方案选型

| 平台 | DRM 方案 | 安全级别 |
|---|---|---|
| Android | Widevine L1（硬件级）/ L3（软件级）| L1 最高，需 TEE |
| iOS/macOS | FairPlay | 系统级保护 |
| 跨平台 | DASH + CENC（Common Encryption）| 同时支持多 DRM |

> **判断 Widevine 等级**：`MediaDrm.isCryptoSchemeSupported(WIDEVINE_UUID)` + 查询 securityLevel

---

## 2. Android Widevine

```kotlin
// ExoPlayer DRM 配置
val drmSessionManagerProvider = DefaultDrmSessionManagerProvider()

val mediaItem = MediaItem.Builder()
    .setUri("https://cdn.example.com/video/manifest.mpd")
    .setDrmConfiguration(
        MediaItem.DrmConfiguration.Builder(C.WIDEVINE_UUID)
            .setLicenseUri("https://license.example.com/widevine")
            .setLicenseRequestHeaders(mapOf(
                "Authorization" to "Bearer $token"  // 鉴权 token
            ))
            .setMultiSession(false)  // 单许可证
            .build()
    )
    .build()
```

### 自定义 License 请求（带用户鉴权）
```kotlin
class AuthenticatedDrmCallback(private val authToken: String) 
    : MediaDrmCallback {
    
    override fun executeProvisionRequest(
        uuid: UUID,
        request: ExoMediaDrm.ProvisionRequest
    ): ByteArray {
        // 设备注册（首次）
        return executePost(request.defaultUrl, request.data, emptyMap())
    }
    
    override fun executeKeyRequest(
        uuid: UUID,
        request: ExoMediaDrm.KeyRequest
    ): Pair<String, ByteArray> {
        val headers = mapOf(
            "Authorization" to "Bearer $authToken",
            "Content-Type" to "application/octet-stream"
        )
        val response = executePost(request.licenseServerUrl, request.data, headers)
        return Pair(request.licenseServerUrl, response)
    }
}
```

---

## 3. iOS FairPlay

```swift
class FairPlayAssetLoaderDelegate: NSObject, AVAssetResourceLoaderDelegate {
    
    func resourceLoader(_ resourceLoader: AVAssetResourceLoader,
                        shouldWaitForLoadingOfRequestedResource request: AVAssetResourceLoadingRequest) -> Bool {
        
        guard let url = request.request.url,
              url.scheme == "skd" else { return false }
        
        // 1. 获取 SPC（Server Playback Context）
        guard let spc = try? request.streamingContentKeyRequestData(
            forApp: appCertificate,
            contentIdentifier: contentId.data(using: .utf8)!,
            options: nil
        ) else { return false }
        
        // 2. 发送 SPC 到 License Server，获取 CKC
        fetchCKC(spc: spc) { ckc in
            // 3. 将 CKC 返回给系统，解密内容
            request.dataRequest?.respond(with: ckc)
            request.finishLoading()
        }
        
        return true
    }
}

// 注册 FairPlay Delegate
let asset = AVURLAsset(url: hlsUrl)
asset.resourceLoader.setDelegate(fairPlayDelegate, queue: DispatchQueue.global())
```

---

## 4. 离线 DRM（下载后播放）

```kotlin
// Android：Widevine 离线 License
val drmConfig = MediaItem.DrmConfiguration.Builder(C.WIDEVINE_UUID)
    .setLicenseUri(licenseUrl)
    .forceSessionsForAudioAndVideoTracks(true)
    .build()

// 下载时同步下载 License
val downloadRequest = DownloadRequest.Builder(id, uri)
    .setKeySetId(offlineLicenseKeySetId)  // 离线 license key
    .build()
```

```swift
// iOS：FairPlay 离线（AVAssetDownloadURLSession）
let config = URLSessionConfiguration.background(withIdentifier: "drm.download")
let session = AVAssetDownloadURLSession(
    configuration: config,
    assetDownloadDelegate: self,
    delegateQueue: .main
)
let task = session.makeAssetDownloadTask(
    asset: asset,
    assetTitle: "Movie Title",
    assetArtworkData: nil,
    options: [AVAssetDownloadTaskMediaSelectionKey: mediaSelection]
)
task?.resume()
```