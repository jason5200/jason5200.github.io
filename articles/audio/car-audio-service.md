# CarAudioService：Zone、音量组、配置文件

> 系列：AAOS-Guide · 06-audio
> 难度：⭐⭐⭐⭐ 深入
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《中间件地图》](../00-overview/middleware.md)

---

手机一份 `AudioManager` 对一个听感；车上要 **分区**（主驾 / 后排娱乐）和 **按 usage 分组调节音量**。这是 **CarAudioService** 的活，配置在 XML，不在 Java 里写死 bus。

焦点仲裁仍然走框架的 `AudioManager.requestAudioFocus`。Car 层决定 **声音进哪条 bus、音量旋钮拧的是哪一组**。

## 一、三层概念

```
Audio Zone（物理/逻辑声区，0 = PRIMARY_AUDIO_ZONE）
    └── Volume Group（旋钮单位，组 ID 在区内从 0 起）
            └── device address（audio_policy 里的 bus）
                    └── context / usage（media、navigation、voice…）
```

14 的分区表在：

- `/vendor/etc/car_audio_configuration.xml`（优先）
- 否则 `/system/etc/car_audio_configuration.xml`

里面的 device `address` 必须在 `audio_policy_configuration.xml` 里存在。Android 14 常用 **configuration version 3**（可 OEM 自定义 context）。组 ID **不跨 Zone 唯一**，API 不带 `zoneId` 时默认主区。

动态路由关掉时，行为接近手机 stream type，座舱产品一般不要关。

## 二、App 怎么拧音量

`CarAudioManager` 多数音量接口是 `@SystemApi`，要 `Car.PERMISSION_CAR_CONTROL_AUDIO_VOLUME`。

```kotlin
val carAudio = car.getCarManager(Car.AUDIO_SERVICE) as CarAudioManager

// 主区、媒体 usage 落在哪个组 —— 不是 getGroupForUsage
val groupId = carAudio.getVolumeGroupIdForUsage(AudioAttributes.USAGE_MEDIA)
val vol = carAudio.getGroupVolume(groupId)
carAudio.setGroupVolume(groupId, vol + 1, 0)
```

带后排 Zone 时用 `getVolumeGroupIdForUsage(zoneId, usage)`、`getGroupVolume(zoneId, groupId)`。

导航播报该进哪只喇叭，由 **配置里 navigation context 绑的 device** 决定，不是 App 选一个 `USAGE` 就能改物理路由。App 要做的是 **usage 标对**，让策略匹配到那条 bus。

## 三、焦点：还是 AudioManager

```kotlin
val attrs = AudioAttributes.Builder()
    .setUsage(AudioAttributes.USAGE_ASSISTANCE_NAVIGATION_GUIDANCE)
    .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
    .build()

val req = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK)
    .setAudioAttributes(attrs)
    .setWillPauseWhenDucked(false)
    .build()

audioManager.requestAudioFocus(req)
// 播完
audioManager.abandonAudioFocusRequest(req)
```

`MAY_DUCK`：音乐降音而不是停。安全音 / 紧急音的 usage 由系统策略抬升，**不要**在媒体 App 里冒充 `USAGE_EMERGENCY`。

注意：部分文档写 `USAGE_NAVIGATION_GUIDANCE`，公开 SDK 里导航 usage 是 **`USAGE_ASSISTANCE_NAVIGATION_GUIDANCE`**。

## 四、和「多媒体」一文的分工

| 问题 | 找谁 |
|------|------|
| 旋钮、分区、bus 配置 | 本文 / `car_audio_configuration.xml` |
| 请求焦点、ducking | `AudioManager` + 本页第三节 |
| HAL 音效、功放 | Audio HAL，不是 Vehicle Property |

倒车雷达「必须压过音乐」是 **usage + 策略 + 独立 bus**，不是 CarPropertyManager 设一个音量属性这么简单。

## 五、调试

```bash
adb shell dumpsys car_service
# 再结合 dumpsys audio、你的 car_audio_configuration.xml
```

改 XML 必须与 `audio_policy_configuration.xml` 的 address 一致，否则服务起不来或组为空。

## 六、总结

| 要点 | 结论 |
|------|------|
| 配置 | `car_audio_configuration.xml` |
| 音量 API | `getVolumeGroupIdForUsage` + `get/setGroupVolume` |
| 焦点 | 仍用 `AudioManager`，usage 要标对 |
| Zone 0 | 主驾区，缺省 API 都打这里 |

---

**下一篇**：[空调：走 CarPropertyManager](car-hvac.md)
