# CarService 架构：从 SystemServer 到车辆服务

> 系列：AAOS-Guide · 01-car-service
> 难度：⭐⭐ 进阶
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《中间件地图》](/articles/00-overview/middleware.md)

---

## 一、它在哪一层

AAOS 比手机 AOSP 多出来的系统服务，入口是 **CarService**。源码主体：

```
packages/services/Car/service/          # 服务实现（跑在 com.android.car）
packages/services/Car/car-lib/          # android.car，给 App 用
packages/services/Car/car-builtin-services/  # SystemServer 里的 Helper
```

启动关系：

```
init → Zygote → system_server
                    │
                    └── CarServiceHelperService
                              │ bindServiceAsUser
                              ▼
                    com.android.car（CarService）
                              │
                              └── ICarImpl
                                    ├── Property / Power / Audio / UX …
                                    └── VehicleStub → IVehicle
```

**CarService 不在 `system_server` 里。** 进程名是 `com.android.car`。Helper 只负责拉起、绑定、在它挂掉时再拉。

## 二、为什么必须独立进程

1. **故障隔离**：VHAL 或某个车辆服务空指针，不应让 AMS/WMS 一起没。行驶中 `system_server` 重启比车载服务重启危险得多。
2. **权限面**：碰车速、车门、空调的调用集中在一个 UID 空间，便于 `enforcePermission`。
3. **生命周期**：可以单独 `kill` / 重启 `com.android.car`，不必重启整个 Android。

## 三、客户端怎么找到它

App 不 bind 某个自定义 Action 去「找车」。标准路径：

```kotlin
val car = Car.createCar(context) ?: return
try {
    val mgr = car.getCarManager(Car.PROPERTY_SERVICE) as CarPropertyManager
    // ...
} finally {
    car.disconnect()
}
```

`Car.createCar` 内部连的是已经 `ServiceManager.addService("car_service", …)` 发布出去的 `ICar`。连失败时先看 CarService 有没有 `onCreate` 成功、VHAL 是否 ready（电源状态机会先 `WAIT_FOR_VHAL`）。

Android 10 之后也有带回调的重载，避免主线程死等。

## 四、ICarImpl 里都有什么

`ICarImpl` 是 `ICar.Stub`，在 `CarService.onCreate()` 里创建并 `init()`。子服务按依赖初始化：先 Vehicle 通路，再 Property，再 Audio / Input 等。

对中间件开发，先认这几块：

| 能力 | App 侧 Manager | 服务侧（概念） | 现状 |
|------|----------------|----------------|------|
| 车辆信号 | `CarPropertyManager` | `CarPropertyService` + Property HAL 封装 | **现行主干** |
| 电源 | `CarPowerManager` | `CarPowerManagementService` | 现行 |
| 音频 Zone / 音量组 | `CarAudioManager` | `CarAudioService` | 现行 |
| 驾驶分心 | `CarUxRestrictionsManager` | `CarUxRestrictionsService` | 现行 |
| 空调 / 信息 / 传感器 Manager | `CarHvacManager` 等 | 多半转到 Property | **已废弃**，不要在新代码用 |

把 HVAC、VIN、车速理解成「不同的 Vehicle Property」，比记三个 Manager 更接近现在的 AOSP。

## 五、一次读信号怎么走

以车速为例（属性 ID 是 `VehiclePropertyIds.PERF_VEHICLE_SPEED`，不是已经不用的 Sensor 常量）：

```
App  CarPropertyManager.getProperty(...)
  → Binder  ICarProperty
  → CarPropertyService：鉴权、查配置、必要时下 HAL
  → VehicleStub / IVehicle.getValues（AIDL，Android 13+）
  → vendor HAL：从网关或 MCU 取物理量
  → 原路返回 CarPropertyValue（含 timestamp、status、areaId）
```

订阅则是 `registerCallback` → 服务侧向 VHAL `subscribe` → `IVehicleCallback` 上报 → 再回调 App。不要在 UI 线程里同步死等 HAL。

## 六、源码怎么啃（自上而下）

1. `car-lib`：`Car.java`、`CarPropertyManager.java`、`VehiclePropertyIds.java`
2. `CarService.java`、`ICarImpl.java`：服务怎么 init、怎么 addService
3. `CarPropertyService` 以及它调用的 HAL 封装类
4. `hardware/interfaces/automotive/vehicle/` 下 AIDL：`IVehicle`、`VehiclePropValue`、`VehicleProperty`

HIDL `IVehicle@2.0` 的 `get()/set()` 只在旧分支上当历史看。本仓库默认 AIDL。

## 七、总结

| 要点 | 结论 |
|------|------|
| 进程 | `com.android.car`，Helper 从 SystemServer bind |
| 门面 | `Car` / `ICar` |
| 信号主干 | Property，不是 Sensor/Hvac Manager |
| 硬件边界 | 只有 VHAL 该懂总线 |

---

**下一篇**：[CarService 启动流程](/articles/car-service/carservice-startup-source.md)
