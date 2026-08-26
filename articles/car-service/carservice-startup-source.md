# CarService 启动：Helper 怎么把 com.android.car 拉起来

> 系列：AAOS-Guide · 01-car-service
> 难度：⭐⭐⭐⭐ 深入
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《CarService 架构》](carservice-architecture.md)

---

文中代码是按 14 标签**结构改写**，不是逐行拷贝。Helper 的包名、`ICarImpl` 构造参数会随分支变，请到 [cs.android.com](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r67:packages/services/Car/) 核对。

## 一、谁在 SystemServer 里动手

手机 AOSP 的 `SystemServer.startOtherServices()` 不会起车辆栈。带 `android.hardware.type.automotive` 的产品才会：

```
SystemServer
  → SystemServiceManager.startService(CarServiceHelperService)
       → bindServiceAsUser(com.android.car)
            → CarService.onCreate()
                 → ICarImpl.init()
                      → VehicleStub / VehicleHal
                      → 各 ICar* 子服务
                 → ServiceManager.addService("car_service", iCar)
```

Helper 源码在 `packages/services/Car/car-builtin-services/`（不要再到 `frameworks/base` 里找一个叫 CarService 的系统服务）。它跑在 **`system_server`**，真正的车辆逻辑在 **`com.android.car`**。

## 二、Helper：bind 而不是 startService 了事

Helper 继承 `SystemService`。`onStart()` 大致做三件事：

1. 组一个指向 `com.android.car` 的 Intent（action 是 Car 的服务接口名）
2. `bindServiceAsUser(..., UserHandle.SYSTEM)`，把独立进程拉起来
3. 在 `ServiceConnection` 里拿到 `ICar`，后续用户切换、崩溃重启也走这里

bind 失败会打 `wtf`：没有这个 APK、feature 配错、SELinux 拦了，车服务都起不来。电源状态机还会停在 `WAIT_FOR_VHAL`，App 侧 `Car.createCar` 一直 null。

## 三、CarService.onCreate：进程入口

`CarService` 是普通 `Service`，跑在自己的进程。`onCreate` 结构接近：

1. 拿到 `IVehicle`（AIDL 默认实例，Java 里常包一层 `VehicleStub`，用来兼容旧 HIDL）
2. `new ICarImpl(...)` —— 这里才是 `ICar.Stub`
3. `mICarImpl.init()` —— 按依赖创建子服务
4. `ServiceManager.addService("car_service", binder)` —— App 的 `Car.createCar` 连的就是它

`onBind` 把同一个 `ICarImpl` 还给 Helper。所以 Helper 和普通 App 看到的是同一份车辆门面，权限在服务内部按 uid 再查。

## 四、ICarImpl.init：先通路，再业务

不要记一张「有 CarHvacService、CarSensorService、CarInfoService」的清单。14 上信号主干已经收进 Property。

更接近现状的顺序：

| 顺序 | 初始化什么 | 为什么先它 |
|------|------------|------------|
| 1 | VehicleStub / VehicleHal | 后面所有 HAL 调用都经过它 |
| 2 | PropertyHalService + CarPropertyService | 属性配置、订阅、权限字符串来自 HAL config |
| 3 | CarPowerManagementService | 必须等 VHAL ready 才能离开 `WAIT_FOR_VHAL` |
| 4 | Audio / Input / UX / Cluster … | 依赖属性或电源已经可用 |

`CarHvacManager` / `CarInfoManager` 在 `car-lib` 里多半是 **deprecated 壳**，底下仍调 Property。新代码不要再按三个独立服务去搜实现。

## 五、VHAL 怎么接到 Java

Android 13+ 默认：

```
IVehicle（AIDL，hwservicemanager / servicemanager 上的 default 实例）
    ↑
VehicleStub
    ↑
VehicleHal / PropertyHalService
    ↑
CarPropertyService
```

旧笔记里的

```java
android.hardware.automotive.vehicle.V2_0.IVehicle.getService();
```

是 **HIDL 2.0**。14 的实现里即便还留兼容，新平台也不该把它当主路径。

Property 初始化时会 `getAllPropConfigs()`，把 HAL 声明的属性表装进 Java。表里没有的 `propId`，App `getProperty` 会失败，这不是 Manager 写错，是 vendor 没配。

## 六、启动失败怎么拆

| 现象 | 更可能的层 |
|------|------------|
| 没有 `com.android.car` 进程 | 镜像不是 automotive、APK 没编进 product |
| 有进程但 `Car.createCar` 失败 | `addService("car_service")` 没跑完、用户未解锁 |
| 有 Car 但属性全 UNAVAILABLE | VHAL 没起来或 config 为空 |
| 卡在开机动画很久 | 电源仍 `WAIT_FOR_VHAL`，HAL 进程 crash 循环 |

```bash
adb shell dumpsys activity services com.android.car
adb shell dumpsys car_service
adb shell dumpsys android.hardware.automotive.vehicle.IVehicle/default
```

## 七、总结

| 要点 | 结论 |
|------|------|
| SystemServer 只放 Helper | 车辆逻辑在 `com.android.car` |
| 拉起方式 | `bindServiceAsUser` |
| 门面 | `ICarImpl` + `ServiceManager.addService("car_service")` |
| 依赖 | 先 Vehicle 通路，再 Property，再电源 / 音频 |

---

**下一篇**：[CarPropertyManager：读写车辆属性](../carservice-api/carproperty-manager.md)
