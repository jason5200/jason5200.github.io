# 权限与驾驶分心：谁能碰车、行驶中 UI 怎么收

> 系列：AAOS-Guide · 08-permission
> 难度：⭐⭐⭐ 进阶
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《CarPropertyManager》](../carservice-api/carproperty-manager.md)

---

车载权限要同时管两件事：**谁能碰哪条车辆信号**，以及 **行驶中屏幕还许干什么**。前者是 `android.car.permission.*` + 属性 config；后者是 `CarUxRestrictionsManager`，不是同一套 API。

## 一、权限从哪来

读/写某个 Vehicle Property 时，CarPropertyService 用 **该属性配置上的权限字符串** 做 `enforcePermission`。字符串在 HAL `VehiclePropConfig` 里，和 `VehiclePropertyIds` 一起对照。

App 侧仍然要在 Manifest 声明，例如：

```xml
<uses-permission android:name="android.car.permission.CAR_SPEED" />
```

`car-lib` 里对应常量是 `Car.PERMISSION_SPEED` 这类，值就是上面那串。

## 二、级别：声明了不等于能用

| 级别 | 典型能力 | 怎么才能有 |
|------|----------|------------|
| normal / dangerous | 读部分状态（如车速） | dangerous 要运行时授权；车企仍可策略拦截 |
| signature | 和平台同签名 | platform key |
| signature\|privileged | 多数「控车」 | 同签名 **或** priv-app + `privapp-permissions-*.xml` 白名单 |

经验上：

- `CAR_SPEED`：读 `PERF_VEHICLE_SPEED`，多为危险权限
- `CONTROL_CAR_CLIMATE`：写 HVAC 设定，特权
- 门 / 窗 / 座椅运动：签名或特权，普通商店应用不要指望

第三方 App `requestPermissions(CONTROL_CAR_CLIMATE)` **不会**变成系统应用。调试控车请用 priv-app 或系统签名包。

## 三、和属性绑定，不要背一张总表

同一 `propId` 读和写可以要不同权限。config 里没有的属性，不是缺权限，是 HAL 没实现。

`dumpsys car_service` 能看到属性列表和权限；对不上时先对 HAL，再对 Manifest。

## 四、驾驶分心是另一条链

```
驾驶状态 / 车速等
    → CarUxRestrictionsService（可配置 XML）
    → CarUxRestrictions
    → App：收视频、禁键盘、限字数
```

查询：

```kotlin
val ux = car.getCarManager(Car.CAR_UX_RESTRICTION_SERVICE) as CarUxRestrictionsManager
val r = ux.currentCarUxRestrictions
if (r.isRequiresDistractionOptimization) {
    val flags = r.activeRestrictions
    val noVideo = flags and CarUxRestrictions.UX_RESTRICTIONS_NO_VIDEO != 0
    val noText = flags and CarUxRestrictions.UX_RESTRICTIONS_NO_TEXT_ENTRY != 0
    // 还有 LIMIT_STRING_LENGTH 等，用 getMaxRestrictedStringLength()
}
```

`isNoVideo()` 这类快捷方法不要当 14 的公开 API 来抄。用 **bitmask**。

监听停车 ↔ 行驶，必须 `registerListener`，页面销毁 `unregisterListener`。只在 `onCreate` 读一次，驶出地库 UI 不会收。

## 五、空调示例：权限 + Property + UX

不要再用 `CarHvacManager`。

```kotlin
if (checkSelfPermission(Car.PERMISSION_CONTROL_CAR_CLIMATE)
    != PackageManager.PERMISSION_GRANTED) {
    // 特权权限：这里弹窗没有用，检查 priv-app / 签名
}

val restrictions = ux.currentCarUxRestrictions
if (restrictions.isRequiresDistractionOptimization) {
    showSimpleClimateUi()  // 大步进 +/- ，不要复杂风口图
} else {
    showFullClimateUi()
}

propertyManager.setProperty(
    java.lang.Float::class.java,
    VehiclePropertyIds.HVAC_TEMPERATURE_SET,
    driverAreaId,   // 来自 config.getAreaIds()，不是写死 0/1/2
    22.5f
)
```

## 六、坑

| 坑 | 处理 |
|----|------|
| 声明了气候权限仍然 SecurityException | 不是 dangerous，要 priv-app 白名单 |
| 模拟器能写、量产不能 | 量产 HAL 写权限更严，或根本不可写 |
| 只做权限、不做 UX | 车企审核 / 合规过不了 |
| UX 和权限混成一个 Manager | Property 管信号，UxRestrictions 管 UI |

更细的 UI 适配见 [CarUxRestrictions](../audio/car-ux.md)。

---

**下一篇**：[CarPowerManagementService](../audio/car-power.md)
