# 音频：MTK HAL 接到 CarAudioService

> 系列：AAOS-Guide · 48-mtk
> 难度：⭐⭐⭐ 进阶
> 更新：2026-08-26
> 前置知识：[《CarAudioService》](/articles/audio/car-audio-service.md)、[《联发科座舱地图》](/articles/mtk/mtk-aaos-map.md)

---

座舱没声、只有导航有声、倒车雷达盖不住音乐，问题可能在 **CarAudio 的 Zone / 音量组**，也可能在 **MTK audio HAL / DSP**。先分清层，再改 XML 或参数。

## 一、两层各管什么

```
App / 导航 / 媒体
  → AudioFlinger 焦点
  → CarAudioService（Zone、volume group、car_audio_configuration.xml）
       → audio HAL（vendor，联发科实现）
            → 内核 sound card / DSP
                 → 功放 / 喇叭 / 咪头
```

[CarAudio 一文](/articles/audio/car-audio-service.md) 讲上半：usage 进哪条 bus、旋钮拧哪一组。  
MTK 这一篇讲下半：HAL 是否把这条 bus **真的送到对应 I2S / DSP 通路**。

`car_audio_configuration.xml` 里的 `device address` 必须在 `audio_policy_configuration.xml` 里存在。联发科工程里后者常和厂商 audio 参数一起改。改了一边没改另一边，就是「策略认为有导航声、功放没动静」。

## 二、常见现象怎么拆

| 现象 | 先疑哪层 |
|------|----------|
| 所有 usage 都没声 | HAL / 时钟 / 功放 mute / DSP 没起来 |
| 只有媒体有、导航无 | CarAudio 路由或 usage 标错 |
| 模拟器正常、实车无声 | 真机 audio 参数、I2S、功放使能 |
| 倒车雷达不够响 | 独立 bus / SAFETY usage，也可能是 DSP 混音增益 |
| 通话有声、媒体无 | 通路切到 downlink，媒体 bus 被关 |

不要用 `CarPropertyManager` 去「设音量属性」代替 `CarAudioManager`，那不是 AAOS 的音量模型。

## 三、联发科侧你通常会碰到的东西

名字随 BSP 变，含义稳定：

- **audio HAL** 实现：`android.hardware.audio` 对应的 vendor 库
- **参数 / 调音**：一堆 audio param、场景表（车企 DSP 调音师改这里）
- **tinyalsa / tinymix**：看控件是否 mute、增益是否为 0
- **DSP firmware**：没 load 就是整机发虚，log 在内核和 vendor 服务

座舱比手机多：**多音区、功放关机、ACC 掉电时序**。ACC 一断 HAL 还在写，容易 NE，去 [AEE](/articles/mtk/mtk-aee.md) 对一下是不是 audio 进程。

## 四、建议的对照实验

1. `dumpsys car_service` 看 Zone / group 是否按你的 XML 建出来  
2. 同一 usage 用 `AudioTrack` 在主区播一段，确认策略层有声  
3. 再 tinymix / 示波器看物理脚  
4. 前两步有、第三步无 → 留在 MTK / 硬件；反过来 → 留在 CarAudio 配置

## 五、和中间件主线的关系

中间件文档默认 **模拟器 + 通用 HAL**。联发科实车要把 **厂商 audio 配置和 `car_audio_configuration.xml` 当成一套** 来维护。本篇不列未公开的 param 字段表。

---

**下一篇**：[中间件地图](/articles/00-overview/middleware.md)（从芯片再回到 CarService）
