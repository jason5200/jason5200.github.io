# CarService 架构：从 SystemServer 到车辆服务

> 系列：AAOS-Guide · 01-car-service
> 难度：⭐⭐ 进阶
> 前置知识：《车载 Android 全景：AAOS 到底是什么》、Android 系统启动流程

---

## 一、CarService 在系统里的位置

在上一篇我们提到：AAOS 比手机 Android 多出来的核心，就是**汽车专属服务栈**，而这条栈的总入口就是 `CarService`。

先定位它在源码里的位置：

```
AOSP 源码路径：packages/services/Car/service/
```

在系统启动流程中，`CarService` 的启动时机是这样的：

```
init 进程
   │
   ▼
Zygote
   │
   ▼
SystemServer（系统服务总进程）
   │
   ├── AMS / WMS / PMS ...      ← 手机 Android 就有的通用服务
   │
   └── CarServiceHelperService  ← AAOS 专属：负责拉起 CarService
           │
           ▼
       CarService（独立进程 com.android.car）
           │
           ├── CarPropertyService     车辆属性
           ├── CarPowerManagementService  电源管理
           ├── CarHvacService         空调控制
           ├── CarInfoService         车辆信息
           ├── CarSensorService       传感器
           └── ...                    更多子服务
```

**关键点**：`CarService` 不是跑在 `SystemServer` 进程里的，它是一个**独立的系统进程**（`com.android.car`），由 `CarServiceHelperService` 负责启动和保活。

## 二、为什么 CarService 要独立成进程

这是理解 AAOS 架构的一个关键问题。三个原因：

1. **隔离故障**：车辆服务一旦崩溃，不能拖垮整个 SystemServer（否则手机式「系统重启」发生在行驶中的车上会很危险）。
2. **权限边界清晰**：车辆硬件操作是高危动作，独立进程便于做权限与访问控制。
3. **可独立重启**：`CarServiceHelperService` 可以单独拉起/重启 CarService，而不影响系统其他部分。

## 三、CarService 的分层结构

CarService 内部不是一坨代码，而是严格分层的：

```
┌────────────────────────────────────────────────────┐
│   Car API（android.car.*，App 直接调用）              │
├────────────────────────────────────────────────────┤
│   CarService（ICarImpl 实现层）                      │
│      ├── CarPropertyService                        │
│      ├── CarPowerManagementService                 │
│      ├── CarHvacService                            │
│      └── ...                                       │
├────────────────────────────────────────────────────┤
│   Vehicle HAL（AIDL 定义，硬件抽象）                 │
│      └── IVehicle / IVehicleCallback               │
├────────────────────────────────────────────────────┤
│   Vehicle HAL 实现（对接 CAN 总线 / ECU / 传感器）    │
└────────────────────────────────────────────────────┘
```

- **Car API**：App 侧，通过 `Car` 入口类拿到各种 Manager（如 `CarPropertyManager`）。
- **CarService**：系统侧，实现这些 Manager 的逻辑。
- **Vehicle HAL**：用 AIDL 定义的接口，是系统与车辆硬件的分界线。

## 四、核心子服务一览

| 子服务 | 职责 | 对应的 App API |
|--------|------|----------------|
| `CarPropertyService` | 读写车辆属性（车速、续航、档位） | `CarPropertyManager` |
| `CarPowerManagementService` | 车辆电源状态机（启动/熄火/休眠） | `CarPowerManager` |
| `CarHvacService` | 空调控制 | `CarHvacManager` |
| `CarInfoService` | 车辆静态信息（VIN、厂商、车型） | `CarInfoManager` |
| `CarSensorService` | 车辆传感器数据 | `CarSensorManager` |
| `CarAudioService` | 车载音频路由 | `CarAudioManager` |
| `CarUxRestrictionsService` | 驾驶分心限制 | `CarUxRestrictionsManager` |

> 这些子服务中，**CarPropertyService 是最基础、最常用的**，下一篇会专门讲它和 `CarPropertyManager`。

## 五、一个请求的完整链路

以「App 读取当前车速」为例，看一次完整调用：

```
App 调用 CarPropertyManager.getIntProperty(VEHICLE_SPEED)
        │
        ▼
Car API（android.car）通过 Binder 跨进程
        │
        ▼
CarPropertyService.getIntProperty()
        │
        ▼
Vehicle HAL（IVehicle.get(VehiclePropConfig)）
        │
        ▼
HAL 实现读 CAN 总线 → 返回车速值
        │
        ▼（原路返回）
App 拿到结果
```

**核心理解**：App 永远不直接碰硬件，每一层都有清晰的职责边界。这就是为什么你能在 App 里用一行代码读车速，而不用管底层 CAN 协议。

## 六、如何阅读 CarService 源码

给初读源码的人一条建议路径：

1. **先看接口**：`packages/services/Car/car-lib` 下的 `Car.java`、`CarPropertyManager.java`，理解 App 视角的 API。
2. **再看服务入口**：`packages/services/Car/service/src/com/android/car/CarService.java` 和 `ICarImpl.java`，看服务如何初始化。
3. **深入一个子服务**：挑 `CarPropertyService` 读透一个（不要贪多），理解它如何封装 Vehicle HAL。
4. **最后看 HAL 定义**：`hardware/interfaces/automotive/vehicle/` 下的 AIDL，理解系统与硬件的契约。

## 七、总结

| 要点 | 结论 |
|------|------|
| CarService 是什么 | AAOS 汽车专属系统服务的总入口 |
| 进程模型 | 独立进程 `com.android.car`，由 CarServiceHelperService 拉起 |
| 分层 | Car API → CarService 子服务 → Vehicle HAL → 硬件 |
| 最基础子服务 | CarPropertyService（车辆属性读写） |
| 学习建议 | 从 App API 到服务到 HAL，自上而下读 |

---

**下一篇预告**：《CarPropertyManager：如何读写车辆属性》

> 配套仓库：[AAOS-Guide](https://github.com/jason5200/AAOS-Guide) · 实战 Demo：[Car-Launcher-Demo](https://github.com/jason5200/Car-Launcher-Demo)
