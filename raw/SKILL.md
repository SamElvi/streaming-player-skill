---
name: streaming-player
description: |
  iOS 和 Android 流媒体播放器开发、架构设计与场景体验优化专家技能。
  
  涵盖：HLS/DASH/RTMP 协议实现、AVPlayer/ExoPlayer 深度调优、首帧加速、卡顿治理、
  画质自适应（ABR）、预加载/预渲染策略、直播低延迟、短视频滑动流畅度、
  DRM 版权保护、弱网对抗、音视频同步、硬件解码、功耗优化。
  
  当用户提到以下任何场景时必须使用本技能：
  播放器开发 / ExoPlayer / AVPlayer / IJKPlayer / VLC / FFmpeg 播放端 /
  HLS DASH RTMP 直播 短视频 长视频 / 首帧优化 卡顿 缓冲 ABR 自适应码率 /
  音视频同步 lip-sync / 预加载 预渲染 / DRM Widevine FairPlay /
  弱网 断网 续播 / 播放器架构 渲染管线 / 抖音/快手/YouTube/Netflix 同类播放体验。
---

# 流媒体播放器开发与体验优化 Skill

## 使用说明

阅读本 SKILL.md 后，根据用户需求选读以下参考文件：

| 参考文件 | 何时阅读 |
|---|---|
| `references/architecture.md` | 播放器整体架构、分层设计、模块划分 |
| `references/protocols.md` | HLS / DASH / RTMP / WebRTC 协议细节 |
| `references/android-exoplayer.md` | Android ExoPlayer 实现与调优 |
| `references/ios-avplayer.md` | iOS AVPlayer / AVFoundation 实现与调优 |
| `references/startup-optimization.md` | 首帧、起播速度优化 |
| `references/smoothness-abr.md` | 卡顿治理、ABR 自适应码率 |
| `references/scenarios.md` | 短视频、直播、长视频典型场景方案 |
| `references/quality-power.md` | 画质增强、功耗、内存优化 |
| `references/drm-security.md` | DRM、内容安全 |
| `references/weak-network.md` | 弱网对抗、断网续播 |

---

## 核心原则

### 1. 分层思考
任何播放问题都从五层定位：**网络层 → 解封装层 → 解码层 → 渲染层 → 业务层**

### 2. 数据驱动
优化必须有指标支撑：
- **起播耗时**（Time-to-First-Frame, TTFF）：< 300ms 优秀，< 800ms 达标
- **卡顿率**（Buffering Ratio）：< 0.3% 直播，< 0.5% 点播
- **秒开率**：短视频 > 95%，长视频 > 90%
- **首帧成功率**、**播放完成率**、**ABR 切换频次**

### 3. 场景驱动差异化方案

| 场景 | 核心诉求 | 关键技术 |
|---|---|---|
| 短视频（抖音/Reels）| 滑动即播、零等待 | 预加载、预渲染、SurfacePool |
| 直播 | 低延迟、抗抖动 | LL-HLS、RTMP、QUIC、GOP 缓存 |
| 长视频（Netflix）| 高画质、省流 | DASH ABR、DRM、HDR、内存分段 |
| 游戏直播 | 超低延迟 | WebRTC、RTMP 推流 |

### 4. 端侧与 CDN 协同
播放器体验不是纯端侧问题，需要联动：
- **CDN 调度**：就近节点、预热、302 调度
- **服务端推流质量**：GOP 长度、关键帧间隔
- **服务端 ABR 决策**辅助（Server-Side ABR）

---

## 快速诊断流程

```
用户反馈播放问题
        │
  ┌─────▼──────┐
  │  黑屏/白屏  │──▶ 参见 startup-optimization.md
  │  首帧慢     │
  └─────┬──────┘
        │
  ┌─────▼──────┐
  │   卡顿/     │──▶ 参见 smoothness-abr.md
  │   转圈缓冲  │
  └─────┬──────┘
        │
  ┌─────▼──────┐
  │  花屏/绿屏  │──▶ 解码器问题，参见平台文件
  │  马赛克     │
  └─────┬──────┘
        │
  ┌─────▼──────┐
  │  音画不同步 │──▶ 参见 architecture.md §sync
  └─────┬──────┘
        │
  ┌─────▼──────┐
  │  弱网/离线  │──▶ 参见 weak-network.md
  └────────────┘
```

---

## 回答规范

1. **先明确平台**（iOS / Android / 跨端），再给针对性方案
2. **代码示例**：Android 用 Kotlin，iOS 用 Swift，关键注释说明 *为什么* 这样做
3. **给出可度量的优化目标**（TTFF、卡顿率等），而非模糊描述
4. **提及风险与权衡**：预加载消耗流量、硬解兼容性碎片化等
5. **区分快速修复与系统优化**：给出短期 / 长期两套方案