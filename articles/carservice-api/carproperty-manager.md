# CarPropertyManager：如何读写车辆属性

> 系列：AAOS-Guide · 02-carservice-api
> 难度：⭐⭐⭐ 实战
> 前置知识：《CarService 架构》

---

## 一、什么是车辆属性（Car Property）

在 AAOS 里，**车辆上一切可以被读写的状态，都抽象成了「车辆属性（Car Property）」**：

- 车速、续航、油量/电量、档位、里程
- 空调温度、风量、车窗、车灯
- 门锁状态、安全带、座椅位置

每个属性都有一个唯一 ID，例如：

| 属性 | 常量（VehiclePropertyIds） |
|------|---------------------------|
| 车速 | `VEHICLE_SPEED` |
| 续航里程 | `EV_BATTERY_LEVEL` / `RANGE_REMAINING` |
| 档位 | `GEAR_SELECTION` |
| 空调温度 | `HVAC_TEMPERATURE_SET` |

**核心理解**：App 不直接碰 CAN 总线，而是通过 `CarPropertyManager` 读写这些属性，由系统层（CarPropertyService → Vehicle HAL）完成与硬件的交互。

## 二、CarPropertyManager 是什么

`CarPropertyManager` 是 App 侧读写车辆属性的入口，属于 `android.car` 包。它通过 `Car` 对象获取：

```kotlin
val car = Car.createCar(context)           // 1. 创建 Car 实例
val carPropertyManager = car.getCarManager(
    Car.PROPERTY_SERVICE
) as CarPropertyManager                    // 2. 拿到 CarPropertyManager
```

> ⚠️ 使用前需在 `AndroidManifest.xml` 声明车辆权限，例如读车速：
> ```xml
> <uses-permission android:name="android.car.permission.CAR_SPEED" />
> ```

## 三、读一个属性（同步 / 异步）

以读车速为例：

```kotlin
// 同步读取（注意：可能阻塞，避免在主线程调用）
val speedProp = carPropertyManager.getProperty(
    Float::class.java,
    VehiclePropertyIds.VEHICLE_SPEED,
    0
)

// 异步读取（推荐）
carPropertyManager.getPropertyCallback(
    Float::class.java,
    VehiclePropertyIds.VEHICLE_SPEED,
    0
).addOnSuccessListener { value ->
    // value.value 就是车速（m/s）
    Log.d(TAG, "当前车速: ${value.value} m/s")
}.addOnFailureListener { e ->
    Log.e(TAG, "读取失败", e)
}
```

**注意点**：
1. 属性有**类型**（int / float / boolean / string 等），读的时候类型要匹配。
2. 第三个参数 `areaId`：多数属性填 `0` 表示全局；多区属性（如分区空调）需要指定区域。
3. 读写车辆属性通常需要**系统签名权限**，普通第三方 App 有限制；系统 App 或具备 `car` 权限的 App 才可访问。

## 四、写一个属性

以设置空调温度为例：

```kotlin
carPropertyManager.setProperty(
    Float::class.java,
    VehiclePropertyIds.HVAC_TEMPERATURE_SET,
    0,
    22.5f
)
```

写操作也有对应的回调版本 `setPropertyCallback`。写之前通常要先确认：
- 该属性**可写**（通过 `VehiclePropertyIds` 对应的权限声明）
- 写入值在**合法范围**内（可用 `getFloatPropertyRange` 查询范围）

## 五、监听属性变化（重要）

车机上最常用的场景是**持续监听**某个属性（比如车速、档位变化），而不是轮询：

```kotlin
val listener = CarPropertyManager.OnPropertyChangedListener { values ->
    values.forEach { value ->
        if (value.propertyId == VehiclePropertyIds.VEHICLE_SPEED) {
            Log.d(TAG, "车速变化: ${value.value}")
        }
    }
}

carPropertyManager.registerCallback(listener, VehiclePropertyIds.VEHICLE_SPEED, 5f)

// 不用时记得注销
// carPropertyManager.unregisterCallback(listener)
```

`registerCallback` 的第三个参数是**采样率**（Hz），表示每秒最多回调几次。这很关键：车速这类高频属性要合理设置采样率，避免刷爆 UI。

## 六、完整链路回顾

结合上一篇《CarService 架构》，一次读车速的完整链路：

```
App: CarPropertyManager.getProperty(VEHICLE_SPEED)
        │  (Binder 跨进程)
        ▼
CarPropertyService.getProperty()
        │  (AIDL)
        ▼
Vehicle HAL: IVehicle.get(VehiclePropConfig)
        │
        ▼
HAL 实现读 CAN 总线 → 返回车速
        │  (原路返回)
        ▼
App 拿到结果 / 回调触发
```

## 七、常见坑与最佳实践

| 坑 | 建议 |
|----|------|
| 在主线程同步读属性 | 用异步 API 或切到工作线程 |
| 类型不匹配导致异常 | 读前确认属性的数据类型 |
| 忘记注销监听器 | `unregisterCallback`，避免内存泄漏 |
| 采样率设太高 | 按需设置，车速 5-10Hz 足够 |
| 权限缺失静默失败 | 检查 `android.car.permission.*` 声明 |

## 八、总结

| 要点 | 结论 |
|------|------|
| 车辆属性是什么 | 车辆硬件状态的抽象，有唯一 ID 和类型 |
| 读写入口 | `CarPropertyManager`（通过 `Car` 获取） |
| 三种操作 | 读（get）、写（set）、监听（registerCallback） |
| 链路 | App → CarPropertyService → Vehicle HAL → 硬件 |
| 关键注意 | 权限、类型匹配、采样率、注销监听 |

---

**下一篇预告**：《车机多屏显示：从 Display 到 Surface 的链路》

> 配套仓库：[AAOS-Guide](https://github.com/jason5200/AAOS-Guide) · 实战 Demo：[Car-Launcher-Demo](https://github.com/jason5200/Car-Launcher-Demo)
