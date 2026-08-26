# 车载中间件地图：从 App 到总线

> 系列：AAOS-Guide · 00-overview
> 难度：⭐⭐ 进阶
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《AAOS 到底是什么》](/articles/00-overview/aaos-intro.md)

---

座舱中间件要解决的问题就一件事：**让应用用 Android 的方式碰车，同时不让应用直接碰 CAN / ECU。**

## 一、一张图先记住

```
应用进程
  android.car（Car / CarPropertyManager / CarAudioManager / …）
        │ Binder（ICar / ICarProperty 等）
        ▼
com.android.car（CarService）
  ICarImpl
    ├── PropertyHalService / CarPropertyService
    ├── CarPowerManagementService
    ├── CarAudioService
    └── CarUxRestrictionsService
        │ AIDL（android.hardware.automotive.vehicle.IVehicle）
        ▼
Vehicle HAL（vendor 实现）
        │ 专有协议 / SOME-IP / CAN
        ▼
车身域控 / 网关 / ECU
```

手机 AOSP 在 SystemServer 里已经有 AMS、WMS；AAOS **多出来的这一截**，才是车载中间件。

## 二、四条必须分清的边界

| 边界 | 谁负责 | 不要做的事 |
|------|--------|------------|
| App ↔ CarService | Binder，走 `android.car` | App 里直接 JNI 读 CAN |
| CarService ↔ VHAL | `IVehicle` AIDL | 在 Java 服务里写死某款 MCU 协议 |
| VHAL ↔ 总线 | vendor HAL | 把车企报文格式泄漏到 CarPropertyManager |
| 权限 / UX | Permission + UxRestrictions | 只声明权限、不处理行驶中 UI |

CarService 做成 **独立进程 `com.android.car`**，就是为了：车辆栈崩了不要拖死 `system_server`，权限面也可以单独收。

## 三、信号总线：Property 才是主干

老代码里能看到 `CarSensorManager`、`CarHvacManager`、`CarInfoManager`。在 Android 10 之后它们陆续废弃。

**现行主干是车辆属性（Vehicle Property）**：

- 每个信号一个 `propId`（`VehiclePropertyIds`，与 HAL 侧 `VehicleProperty` 对应）
- 可带 `areaId`（主驾 / 副驾空调分区等）
- 配置里写明类型、读写、权限、变化模式（静态 / 连续 / 按变更）

读车速、设空调、问 VIN，最后都进 `CarPropertyService` → VHAL。电源状态、音频路由、驾驶分心是**另一类中间件**，不要全塞进 Property。

## 四、和「系统服务」怎么对应

| 你要做的事 | 中间件入口 | 服务进程内 |
|------------|------------|------------|
| 读/写/订阅车辆信号 | `Car.getCarManager(Car.PROPERTY_SERVICE)` | `CarPropertyService` |
| 熄火保存、挂起恢复 | `Car.POWER_SERVICE` | `CarPowerManagementService` |
| 分区音量、车载焦点 | `Car.AUDIO_SERVICE` | `CarAudioService` |
| 行驶中收 UI | `Car.CAR_UX_RESTRICTION_SERVICE` | `CarUxRestrictionsService` |
| 拿到车 | `Car.createCar(context)` | `ICar` / `ICarImpl` |

`Car` 是客户端门面，不是服务本身。连不上时先查：`com.android.car` 是否起来、VHAL 是否 `default` 实例 ready。

## 五、调试时从哪一层下手

| 现象 | 先看 |
|------|------|
| App 拿不到 `Car` | `dumpsys activity services com.android.car`、logcat `CarService` |
| getProperty 抛 SecurityException | 属性对应的 `android.car.permission.*`、是否 priv-app / 签名 |
| 值永远是 0 / UNAVAILABLE | VHAL 是否订阅成功、vendor 是否上报 `AVAILABLE` |
| 模拟器没有真车 | `dumpsys car_service`，用 car_service 的 inject 往 VHAL 灌事件 |

```bash
adb shell dumpsys car_service
adb shell dumpsys android.hardware.automotive.vehicle.IVehicle/default
```

## 六、建议阅读顺序

1. [CarService 架构](/articles/car-service/carservice-architecture.md) — 进程与子服务
2. [CarPropertyManager](/articles/carservice-api/carproperty-manager.md) — 先把客户端 API 用对
3. [Vehicle HAL](/articles/permission/vehicle-hal.md) — 再看契约
4. [CarPropertyService](/articles/car-service/carproperty-source.md) — 中间这一跳
5. [权限与分心](/articles/permission/car-permission.md) · [电源](/articles/audio/car-power.md) · [音频](/articles/audio/car-audio-service.md)

---

**下一篇**：[CarService 架构：从 SystemServer 到车辆服务](/articles/car-service/carservice-architecture.md)
