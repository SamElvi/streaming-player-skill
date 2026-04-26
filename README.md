# 📺 streaming-player-skill

> iOS & Android 流媒体播放器开发与体验优化专家知识库
> 覆盖短视频、直播、长视频全场景，对标抖音、快手、YouTube、Netflix 工业级实践。

---

## 👤 作者

**XUGB**
高级产品经理 · 音视频平台 / 基础设施 / 全球内容分发

9 年音视频基础设施产品经验，现为独立产品人。


---

## 🎯 Skill 定位

本 Skill 将作者在字节跳动 PICO、快手等平台积累的**工业级流媒体播放器**实战经验系统化，帮助 开发者 在面对播放器开发与体验优化问题时给出专业、可落地的方案。

**适用场景：**

| 场景 | 关键词 |
|---|---|
| 播放器开发 | ExoPlayer / AVPlayer / IJKPlayer / FFmpeg |
| 协议与格式 | HLS / DASH / RTMP / WebRTC / fMP4 / TS |
| 起播优化 | 首帧 / TTFF / 秒开率 / 预加载 / 预渲染 |
| 流畅度治理 | 卡顿 / 缓冲 / ABR 自适应码率 / 追帧 |
| 弱网与稳定性 | 断网续播 / 多 CDN 切换 / QUIC |
| 内容安全 | DRM / Widevine / FairPlay / 好莱坞 L1 |
| 典型产品场景 | 短视频滑动流畅度 / 直播低延迟 / 长视频高画质 |

---

## 📦 文件结构

```
streaming-player-skill/
├── SKILL.md                        # 核心：原则、诊断流程、回答规范
└── references/
    ├── architecture.md             # 播放器分层架构、线程模型、A/V Sync
    ├── protocols.md                # HLS/DASH/RTMP/WebRTC 协议详解
    ├── startup-optimization.md     # 首帧 / TTFF 优化全攻略
    ├── smoothness-abr.md           # 卡顿治理 & ABR 自适应码率
    ├── scenarios.md                # 短视频 / 直播 / 长视频场景方案
    ├── weak-network.md             # 弱网对抗、断网续播、DRM
    └── quality-power.md            # 画质增强、功耗、内存优化
```

---

## 🚀 快速上手

将 `streaming-player-skill.skill` 文件安装到 Claude 技能库后，直接在对话中提出播放器相关问题即可触发。

**示例提问：**
```
"ExoPlayer 首帧太慢，怎么优化？"
"帮我设计一个短视频播放器池"
"直播卡顿率高，从哪里排查？"
"iOS AVPlayer 怎么接入 FairPlay DRM？"
"DASH ABR 算法如何选择码率？"
```

---

## 🏗️ 技术覆盖范围

**Android**：ExoPlayer 3.x · MediaCodec · SurfaceView/TextureView · Widevine DRM · OkHttp + HTTPDNS

**iOS**：AVPlayer / AVFoundation · VideoToolbox · Metal 渲染 · FairPlay DRM · LL-HLS

**跨平台**：FFmpeg · IJKPlayer · VLC · QUIC/HTTP3

**协议**：HLS · LL-HLS · DASH · RTMP · HTTP-FLV · WebRTC

**编解码**：H.264 · H.265/HEVC · AV1 · VP9 · MV-HEVC（VR）

---

