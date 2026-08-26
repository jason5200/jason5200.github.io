# CarPropertyManager：读写车辆属性

> 系列：AAOS-Guide · 02-carservice-api
> 难度：⭐⭐⭐ 实战
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《CarService 架构》](../car-service/carservice-architecture.md)

---

车辆上能被系统建模的状态，在 AAOS 里叫 **Vehicle Property**：车速、电量、档位、HVAC 设定、VIN 等。App **只**通过 `CarPropertyManager` 碰它们。

下面的 API 以 `android.car` 14 为准。网上不少示例仍写 `VEHICLE_SPEED`、`getPropertyCallback().addOnSuccessListener`，那不是这个 Manager 的接口。

## 一、常用属性 ID

定义在 `android.car.VehiclePropertyIds`，与 HAL 的 `VehicleProperty` 对齐。

| 含义 | 常量 | 常见类型 |
|------|------|----------|
| 车速 | `PERF_VEHICLE_SPEED` | `Float`，单位 m/s |
| 选档 | `GEAR_SELECTION` | `Integer` |
| 续航 | `RANGE_REMAINING` | `Float`，米 |
| 动力电池 | `EV_BATTERY_LEVEL` | `Float` |
| 空调设定温度 | `HVAC_TEMPERATURE_SET` | `Float`，摄氏 |
| 车厂 / 车型 | `INFO_MAKE` / `INFO_MODEL` | `String` |

全局属性 `areaId` 用 `0`。分区空调要用对应 Area（主驾座椅、左前等），先 `getPropertyList()` 看 `CarPropertyConfig.getAreaIds()`。

## 二、拿到 Manager

```kotlin
val car = Car.createCar(context)
if (car == null) {
    // com.android.car 未起来，或当前用户没有车服务
    return
}
val propertyManager = car.getCarManager(Car.PROPERTY_SERVICE) as CarPropertyManager
```

Manifest：

```xml
<uses-permission android:name="android.car.permission.CAR_SPEED" />
```

只声明不够：很多写属性是 `signature|privileged`，普通商店应用拿不到。调试先用系统签名或 priv-app。

用完 `car.disconnect()`，避免泄漏连接。

## 三、读：getProperty

```kotlin
val speed = propertyManager.getProperty(
    java.lang.Float::class.java,
    VehiclePropertyIds.PERF_VEHICLE_SPEED,
    /* areaId */ 0
)
if (speed != null && speed.status == CarPropertyValue.STATUS_AVAILABLE) {
    val mps = speed.value as Float
    val kmh = mps * 3.6f
}
```

要点：

- 类型必须和配置一致，否则抛 `IllegalArgumentException`。
- 看 `CarPropertyValue.status`，不要把 `UNAVAILABLE` 当 0 速。
- `getProperty` 会进 Binder；不要在主线程高频同步读。连续变化用订阅。

## 四、写：setProperty

```kotlin
propertyManager.setProperty(
    java.lang.Float::class.java,
    VehiclePropertyIds.HVAC_TEMPERATURE_SET,
    areaId,
    22.5f
)
```

写之前用 `getCarPropertyConfig(propId)` 确认 `isWritable()`，以及 min/max。HVAC 通常要 `android.car.permission.CONTROL_CAR_CLIMATE`（特权级）。

## 五、订阅：CarPropertyEventCallback

这是车上最常用的路径。

```kotlin
private val speedCallback = object : CarPropertyManager.CarPropertyEventCallback {
    override fun onChangeEvent(value: CarPropertyValue<*>) {
        if (value.propertyId != VehiclePropertyIds.PERF_VEHICLE_SPEED) return
        if (value.status != CarPropertyValue.STATUS_AVAILABLE) return
        val mps = value.value as Float
        // 更新 UI：切回主线程
    }

    override fun onErrorEvent(propId: Int, areaId: Int) {
        // 配置没有、HAL 拒绝、权限中途被撤
    }
}

propertyManager.registerCallback(
    speedCallback,
    VehiclePropertyIds.PERF_VEHICLE_SPEED,
    CarPropertyManager.SENSOR_RATE_UI  // 5 Hz 量级，够表盘
)

// 页面销毁：
propertyManager.unregisterCallback(speedCallback)
```

采样率常量（float，单位 Hz）：

| 常量 | 约值 | 用途 |
|------|------|------|
| `SENSOR_RATE_ONCHANGE` | 0 | 只在变更时 |
| `SENSOR_RATE_NORMAL` | 1 | 低速状态 |
| `SENSOR_RATE_UI` | 5 | 仪表 / 卡片 |
| `SENSOR_RATE_FAST` | 10 | 更跟手 |
| `SENSOR_RATE_FASTEST` | 100 | 几乎只给内部，App 慎用 |

连续类型属性（车速）不订阅、去 while 循环 `getProperty`，会打满 Binder 和 VHAL。

## 六、和 HAL 的对应关系

```
App  get/set/registerCallback
  → CarPropertyService（鉴权、config、缓存）
  → IVehicle.getValues / setValues / subscribe   （Android 13+ AIDL）
  → vendor HAL
```

旧 HIDL 文档里的 `IVehicle.get(VehiclePropValue)` 不要直接抄到 14 的实现里。

## 七、坑

| 坑 | 处理 |
|----|------|
| 属性 ID 写错成 `VEHICLE_SPEED` | 用 `PERF_VEHICLE_SPEED` |
| 车速当 km/h 显示 | HAL 给的是 m/s |
| 不看 `status` | 无信号时会显示假 0 |
| 不 unregister | 泄漏、熄火后还在回调 |
| 第三方 App 写车门 | 权限模型不允许，不是 API 写错 |

---

**下一篇**：[CarPropertyService 属性通路](../car-service/carproperty-source.md)
