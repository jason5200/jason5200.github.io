# 在智能座舱里写代码

> **车载 Android · Android Framework · AI 应用落地**
>
> 从源码到座舱。Demo 会标明能跑什么、还只是骨架。

---

## 最新文章

| 日期 | 文章 | 系列 |
|------|------|------|
| 2026-08-26 | [车载中间件地图：从 App 到总线](articles/00-overview/middleware.md) | 车载 |
| 2026-08-26 | [CarPropertyManager：读写车辆属性](articles/carservice-api/carproperty-manager.md) | 车载 |
| 2026-08-26 | [Vehicle HAL：AIDL 契约](articles/permission/vehicle-hal.md) | 车载 |
| 2026-08-26 | [CarService 架构：独立进程](articles/car-service/carservice-architecture.md) | 车载 |
| 2026-08-26 | [电源：熄火准备与 completion](articles/audio/car-power.md) | 车载 |

→ [查看全部文章](archive.md)

## 系列导航

| 系列 | 说明 | 篇数 |
|------|------|------|
| [车载 Android（AAOS）](series/aaos.md) | 中间件：HAL → CarService → Property / Power / Audio | **29 篇（主线）** |
| [Android Framework](series/framework.md) | Binder → Handler → AMS/WMS → View | **48 篇** |
| [AI 上车](series/ai.md) | 端侧推理 → 语音 → Agent；其余为选读 | **41 篇（含选读）** |
| **车载实战** | [Launcher 骨架](https://github.com/jason5200/Car-Launcher-Demo) · [对话 Demo（可接 OpenAI 兼容 API）](https://github.com/jason5200/AI-Android-Demo) | 已开源 |

源码叙述默认对照 AOSP `android-14.0.0_r67`。

## 开源项目

- [AAOS-Guide](https://github.com/jason5200/AAOS-Guide) —— 车载中间件学习路线（Vehicle HAL / CarService / 车辆属性）
- [Framework-Source-Note](https://github.com/jason5200/Framework-Source-Note) —— Framework 源码笔记
- [Car-Launcher-Demo](https://github.com/jason5200/Car-Launcher-Demo) —— 车机 Home 骨架
- [AI-Android-Demo](https://github.com/jason5200/AI-Android-Demo) —— 对话 UI + OpenAI 兼容流式接口

## 关于我

车载 Android 系统开发者，写智能座舱、Framework 与端侧 AI。

- GitHub：[jason5200](https://github.com/jason5200)
- 邮箱：[rdszdl@163.com](mailto:rdszdl@163.com)

欢迎 Issue / PR 纠错，尤其是 AOSP 版本和已废弃 API。
