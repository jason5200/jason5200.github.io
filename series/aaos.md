# 车载 Android（AAOS）系列导读

> 主线是**座舱中间件**：Vehicle HAL → CarService → 车辆属性 / 电源 / 音频 / 驾驶分心。
>
> 源码默认对照 AOSP [`android-14.0.0_r67`](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)。大模型内容见 [AI 上车](ai.md)。

---

## 学习路线

```
Binder / 系统服务基础
        │
        ▼
00  全景：AAOS ≠ Android Auto
        │
        ▼
    中间件地图（推荐先读）
        │
        ├── Vehicle HAL     AIDL IVehicle
        ├── CarService      独立进程 com.android.car
        ├── Property        读 / 写 / 订阅
        ├── Power           上电 / 熄火准备 / 挂起
        ├── Audio           Zone + 音量组
        └── Permission/UX   谁能控车、行驶中 UI
```

建议顺序：全景 → [中间件地图](../articles/00-overview/middleware.md) → 架构 / 启动 → CarPropertyManager → HAL → 权限 / 电源 / 音频。

## 文章目录

| 序号 | 文章 |
|------|------|
| 00 | [车载 Android 全景：AAOS 到底是什么](../articles/00-overview/aaos-intro.md) |
| — | [车载中间件地图](../articles/00-overview/middleware.md) |
| 01 | [CarService 架构](../articles/car-service/carservice-architecture.md) |
| 02 | [CarService 启动：Helper 拉起 com.android.car](../articles/car-service/carservice-startup-source.md) |
| 03 | [CarPropertyManager：读写车辆属性](../articles/carservice-api/carproperty-manager.md) |
| 04 | [CarPropertyService：鉴权后再下 HAL](../articles/car-service/carproperty-source.md) |
| 05 | [电源：上电、熄火准备、挂起](../articles/audio/car-power.md) |
| 06 | [空调：走 CarPropertyManager](../articles/audio/car-hvac.md) |
| 07 | [传感器 API 已迁移](../articles/audio/car-sensor.md) |
| 08 | [车辆信息也走 Property](../articles/audio/car-info.md) |
| 09 | [驾驶分心 UX Restrictions](../articles/audio/car-ux.md) |
| 10 | [权限与驾驶分心](../articles/permission/car-permission.md) |
| 11 | [Vehicle HAL：AIDL 契约](../articles/permission/vehicle-hal.md) |
| 12 | [多屏：Display 到 Surface](../articles/multi-display/multi-display.md) |
| 13 | [Cluster 仪表盘](../articles/multi-display/cluster.md) |
| 14 | [HUD](../articles/multi-display/hud.md) |
| 15 | [倒车影像与环视](../articles/perf/reverse-camera.md) |
| 16 | [CarAudioService：Zone 与音量组](../articles/audio/car-audio-service.md) |
| 17 | [多媒体：音频焦点与分区](../articles/permission/multimedia.md) |
| 18 | [蓝牙电话 HFP](../articles/audio/bluetooth-hfp.md) |
| 19 | [导航](../articles/ota/navigation.md) |
| 20 | [OTA 与 A/B 分区](../articles/ota/ota-upgrade.md) |
| 21 | [启动到座舱可用](../articles/perf/boot-process.md) |
| 22 | [冷启动优化](../articles/perf/cold-start.md) |
| 23 | [低内存优化](../articles/perf/memory-optimization.md) |
| 24 | [V2X（概述）](../articles/ota/v2x.md) |
| 25 | [功能安全 ISO 26262（概述）](../articles/ota/functional-safety.md) |
| 26 | [CAN 总线安全（概述）](../articles/ota/can-security.md) |
| 27 | [模拟器与 HIL](../articles/perf/testing.md) |

部分文件还在历史目录（如 `audio/`、`permission/`）下，侧边栏已按主题归类。

## 学习建议

1. 先读完 00 → 11，建立「App → CarService → VHAL」心智模型。`CarHvacManager` / `CarSensorManager` / `CarInfoManager` 不要用在新代码。
2. 做 Launcher / 多屏再读 12–15；做语音/多媒体再读 16–18。
3. 标注「概述」的篇章是背景，不是实车 / HIL 报告。文中代码是按 AOSP 14 **结构改写**，请到 cs.android.com 核对。
4. 动手：[Car-Launcher-Demo](https://github.com/jason5200/Car-Launcher-Demo)（需要 AAOS 模拟器）。

## 配套资源

- 仓库：[AAOS-Guide](https://github.com/jason5200/AAOS-Guide)
- Demo：[Car-Launcher-Demo](https://github.com/jason5200/Car-Launcher-Demo)
