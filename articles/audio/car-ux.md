# CarUxRestrictions：用 bitmask 收行驶中 UI

> 系列：AAOS-Guide · 01-car-service
> 难度：⭐⭐⭐ 进阶
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《权限与驾驶分心》](/articles/permission/car-permission.md)

---

行驶中禁视频、禁键盘，不是 App 自己猜车速，而是 **CarUxRestrictionsService** 根据配置（可含车速、档位、停车状态）算出一份 `CarUxRestrictions`，再发给监听者。

权限管「能不能调 HAL」；UX 管「屏幕上许不许出这块 UI」。两件事都要做。

## 一、先问要不要优化

```kotlin
val mgr = car.getCarManager(Car.CAR_UX_RESTRICTION_SERVICE) as CarUxRestrictionsManager
val r = mgr.currentCarUxRestrictions

if (!r.isRequiresDistractionOptimization) {
    showParkedUi()
    return
}

val flags = r.activeRestrictions
```

`isRequiresDistractionOptimization == true` 时再看具体 bit。不要发明 `isNoVideo()` 这种 14 公开 API。

## 二、常用 restriction 位

以 `CarUxRestrictions` 常量为准（名字稳定，数值不必背）：

| 常量 | App 该做什么 |
|------|----------------|
| `UX_RESTRICTIONS_NO_VIDEO` | 停播、藏播放器 |
| `UX_RESTRICTIONS_NO_TEXT_ENTRY` | 禁用输入框 |
| `UX_RESTRICTIONS_NO_KEYBOARD` | 不弹键盘 |
| `UX_RESTRICTIONS_LIMIT_STRING_LENGTH` | 用 `getMaxRestrictedStringLength()` 截断 |
| `UX_RESTRICTIONS_NO_SETUP` | 藏复杂设置 |
| `UX_RESTRICTIONS_LIMIT_CONTENT` | 限制列表深度 / 条数（见 `getMaxContentDepth` 等） |

```kotlin
fun restricted(r: CarUxRestrictions, mask: Int) =
    (r.activeRestrictions and mask) != 0

if (restricted(r, CarUxRestrictions.UX_RESTRICTIONS_NO_VIDEO)) {
    videoButton.visibility = View.GONE
    stopPlayback()
}
```

配置因车企 XML 而异，**不要假设**「车速 > 0 就一定 NO_VIDEO」。以回调里的 flags 为准。

## 三、必须监听

```kotlin
val listener = CarUxRestrictionsManager.OnUxRestrictionsChangedListener { restrictions ->
    applyUx(restrictions)
}
mgr.registerListener(listener)
// onDestroy:
mgr.unregisterListener(listener)
```

只读一次：地下车库起步后视频还在播。

## 四、产品原则

| 允许（通常） | 禁止（行驶中常见） |
|--------------|-------------------|
| 导航语音、简单切曲、大按钮调温 | 视频、打字搜地点、深层设置 |

语音能否用看 `NO_VOICE` 等 bit，不要和音频焦点混为一谈。

服务侧映射规则在 CarService 的 UX restriction 配置里，改策略是系统镜像的事，不是每个 App 自己读 `PERF_VEHICLE_SPEED` 做一遍。

---

**下一篇**：[Vehicle HAL](/articles/permission/vehicle-hal.md)
