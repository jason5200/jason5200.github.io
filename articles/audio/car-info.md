# 车辆静态信息：VIN / 车型也是 Property

> 系列：AAOS-Guide · 01-car-service
> 难度：⭐⭐ 进阶
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《CarPropertyManager》](/articles/carservice-api/carproperty-manager.md)

---

> **API 状态**  
> `CarInfoManager` / `Car.INFO_SERVICE` 已废弃。VIN、品牌、年款是 **STATIC** 的 Vehicle Property，用 `CarPropertyManager.getProperty` 读一次即可，一般不必订阅。

静态 vs 动态只是 **change mode** 不同，不是两套服务。

## 一、常用 ID

| 常量 | 类型 | 说明 |
|------|------|------|
| `INFO_VIN` | String | 车辆识别码 |
| `INFO_MAKE` | String | 厂商 |
| `INFO_MODEL` | String | 车型 |
| `INFO_MODEL_YEAR` | Int | 年款 |
| `INFO_FUEL_CAPACITY` | Float | 油箱容量等（以 HAL 为准） |
| `INFO_DRIVER_SEAT` | Int | 方向盘位置（左/右舵），area 为 GLOBAL |

这些是 GLOBAL 属性，`areaId` 用 `0`。权限因属性而异，VIN 往往比车型更严。

## 二、读法

```kotlin
val pm = car.getCarManager(Car.PROPERTY_SERVICE) as CarPropertyManager

val vinVal = pm.getProperty(
    String::class.java,
    VehiclePropertyIds.INFO_VIN,
    /* areaId */ 0
)
val vin = vinVal?.takeIf { it.status == CarPropertyValue.STATUS_AVAILABLE }?.value as? String

val yearVal = pm.getProperty(
    Integer::class.java,
    VehiclePropertyIds.INFO_MODEL_YEAR,
    0
)
```

旧 `infoManager.getCarInfo(CarInfoManager.VIN)` 不要出现在新代码里。

## 三、用途

- 云端绑车、诊断：用 VIN，注意隐私与权限
- 功能开关：优先读 **能力相关 Property**（有没有座椅加热），不要只字符串匹配车型名
- 适配 HAL 矩阵：以 `getPropertyList()` / config 为准，车型名只是辅助

## 四、和动态量的区别

| | 静态 INFO_* | 车速等 |
|--|-------------|--------|
| change mode | STATIC | CONTINUOUS / ON_CHANGE |
| 建议 | get 一次 | registerCallback |
| Manager | 都是 CarPropertyManager | 同左 |

---

**下一篇**：[驾驶分心 UX Restrictions](/articles/audio/car-ux.md)
