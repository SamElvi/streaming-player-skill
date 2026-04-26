# 流媒体协议详解

## HLS（HTTP Live Streaming）

### 基本结构
```
Master Playlist (.m3u8)
  ├── 1080p/index.m3u8  (4Mbps)
  ├── 720p/index.m3u8   (2Mbps)
  └── 360p/index.m3u8   (500Kbps)
        └── seg001.ts, seg002.ts, ...（每段 2~6s）
```

### 关键参数
- **分片时长**：点播推荐 6s，直播推荐 2s（LL-HLS 0.5~1s）
- **关键帧间隔**：≤ 分片时长，否则 ABR 切换有残影
- **#EXT-X-PLAYLIST-TYPE**：VOD（点播）/ EVENT（直播回放）/ 无（直播）

### LL-HLS（Low-Latency HLS）
```
#EXT-X-PART-INF:PART-TARGET=0.5     ← 部分分片 0.5s
#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,CAN-SKIP-UNTIL=18
#EXT-X-PART:DURATION=0.5,URI="seg1_part1.mp4"  ← 发布部分分片
```

---

## DASH（Dynamic Adaptive Streaming over HTTP）

### MPD 结构
```xml
<MPD type="dynamic" minimumUpdatePeriod="PT2S">
  <Period>
    <AdaptationSet mimeType="video/mp4" codecs="avc1.640028">
      <Representation id="1" bandwidth="4000000" width="1920" height="1080">
        <SegmentTemplate media="video_$Number$.m4s" duration="4"/>
      </Representation>
      <Representation id="2" bandwidth="2000000" width="1280" height="720"/>
    </AdaptationSet>
  </Period>
</MPD>
```

### fMP4 vs TS
| 特性 | fMP4 | TS |
|---|---|---|
| 随机访问 | 高效（moov box）| 需扫描 |
| ABR 切换 | 无缝 | 需关键帧对齐 |
| 兼容性 | 现代浏览器 | 极广 |
| 直播适用 | 是（cmaf）| 是 |

---

## RTMP（Real-Time Messaging Protocol）

- **用途**：主播推流到服务器（上行）；部分直播观看（下行，逐渐被 FLV over HTTP 替代）
- **延迟**：1~3s
- **端口**：1935（常被防火墙拦截，备选 443/RTMPS）

### 关键参数（推流端）
```
GOP = 2s（关键帧间隔，影响服务端 GOP 缓存和接入延迟）
视频码率 = 1500Kbps（720p 标准）
音频码率 = 128Kbps AAC-LC
```

---

## WebRTC（超低延迟）

- **延迟**：< 200ms
- **适用**：连麦、游戏直播实时互动、视频会议
- **复杂度**：高（需 STUN/TURN 服务器、ICE 协商）

### 与 RTMP 对比
| | RTMP | WebRTC |
|---|---|---|
| 延迟 | 1~3s | < 200ms |
| 适用规模 | 百万级观众 | 千级（点对点）|
| 部署复杂度 | 低 | 高 |
| 音视频质量控制 | 固定 | 自适应（BWE）|

---

## 编解码格式选型

| 格式 | 压缩率 | 硬解支持 | 适用场景 |
|---|---|---|---|
| H.264/AVC | 基准 | 全平台 | 通用，兼容性最好 |
| H.265/HEVC | 好 40% | iOS 全支持，Android 碎片化 | 4K / 省流 |
| AV1 | 好 50% | Android 10+部分，iOS 17+ | 未来主流 |
| VP9 | 好 35% | Android 全支持，iOS Safari 部分 | YouTube |

### 实践建议
- 短视频：H.264 + H.265 双码流，根据设备选择
- 4K 内容：H.265（iOS 优先）/ H.264 降级
- 新项目 2024+：准备 AV1 支持路径