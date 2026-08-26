# 空调控制：用 Property，不要再用 CarHvacManager

> 系列：AAOS-Guide · 01-car-service
> 难度：⭐⭐⭐ 进阶
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《CarPropertyManager》](/articles/carservice-api/carproperty-manager.md)

---

> **API 状态**  
> `CarHvacManager` / `Car.HVAC_SERVICE` 已废弃。空调是一组 **Vehicle Property**，读写走 `CarPropertyManager`。服务进程里也没有一条独立于 Property 的「空调总线」。

HVAC（采暖 / 通风 / 空调）在座舱里控件多，但中间件模型很简单：每个开关、设定值一个 `propId`，分区用 `areaId`。

## 一、常用属性（14）

| 常量 | 类型 | 含义 |
|------|------|------|
| `HVAC_TEMPERATURE_SET` | Float，°C | 设定温度 |
| `HVAC_TEMPERATURE_CURRENT` | Float，°C | 实测温度（不是 `HVAC_TEMPERATURE_VALUE`） |
| `HVAC_FAN_SPEED` | Int | 风速档 |
| `HVAC_FAN_DIRECTION` | Int | 吹脚 / 吹脸等 |
| `HVAC_AC_ON` | Boolean | 压缩机请求 |
| `HVAC_RECIRC_ON` | Boolean | 内循环 |
| `HVAC_POWER_ON` | Boolean | 空调系统电源 |
| `HVAC_TEMPERATURE_DISPLAY_UNITS` | Int | 界面用 °C / °F，设定值仍以 HAL 单位为准 |

写设定需要 `android.car.permission.CONTROL_CAR_CLIMATE`（特权级）。行驶中 UI 还要听 [UxRestrictions](/articles/audio/car-ux.md)。

## 二、areaId 不是 0=主驾、1=副驾

HVAC 属性的 VehicleArea 是 **SEAT**。`areaId` 是 `VehicleAreaSeat` 的 **bit 组合**，由 **这台车的 HAL config** 决定，例如：

- 主驾：`VehicleAreaSeat.SEAT_ROW_1_LEFT`（左舵）
- 副驾：`SEAT_ROW_1_RIGHT`
- 双区绑在一起：`LEFT | RIGHT` 作为一个 areaId

**不要写死 0、1、2。** 正确做法：

```kotlin
val cfg = propertyManager.getCarPropertyConfig(VehiclePropertyIds.HVAC_TEMPERATURE_SET)
val areaIds = cfg?.areaIds ?: intArrayOf()
// 用 areaIds[i] 去 set/get
```

全局属性才用 `areaId = 0`。把主驾温度 set 到 0，在分区车上经常直接失败。

## 三、读写示例

```kotlin
val pm = car.getCarManager(Car.PROPERTY_SERVICE) as CarPropertyManager
val area = areaIds.first() // 先 dump 再选主驾 bit

pm.setProperty(
    java.lang.Float::class.java,
    VehiclePropertyIds.HVAC_TEMPERATURE_SET,
    area,
    22.5f
)

val set = pm.getProperty(
    java.lang.Float::class.java,
    VehiclePropertyIds.HVAC_TEMPERATURE_SET,
    area
)
```

风速是 Int：`getProperty(Integer::class.java, HVAC_FAN_SPEED, area)`。不要对 Float 属性调 `setIntProperty`。

订阅设定变化用 `CarPropertyEventCallback` + `SENSOR_RATE_ONCHANGE`，和车速那套一样。

## 四、单位

`HVAC_TEMPERATURE_SET` 在 VHAL 文档里单位是 **摄氏**。显示华氏只影响 UI。`HVAC_TEMPERATURE_DISPLAY_UNITS` 用来同步界面单位，不要把设定值按华氏写进 SET。

## 五、和旧 CarHvacManager 的对照

| 旧写法 | 现在 |
|--------|------|
| `car.getCarManager(Car.HVAC_SERVICE)` | `Car.PROPERTY_SERVICE` |
| `setIntProperty(HVAC_TEMPERATURE_SET, 0, 24)` | `setProperty(Float.class, …, 22.5f)` + 真 areaId |
| 独立 CarHvacService | PropertyHalService |

读历史代码时看到 HvacManager，把它翻译成 Property 即可。

---

**下一篇**：[车辆信息也走 Property](/articles/audio/car-info.md)
