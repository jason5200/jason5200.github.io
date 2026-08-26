# Vehicle HAL：AIDL 契约和 vendor 实现

> 系列：AAOS-Guide · 12-vehicle-hal
> 难度：⭐⭐⭐⭐ 深入
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《中间件地图》](/articles/00-overview/middleware.md)、[《CarPropertyManager》](/articles/carservice-api/carproperty-manager.md)

---

CarService **不准**直接解析 CAN。和车的边界是 **Vehicle HAL**：AIDL 接口 `android.hardware.automotive.vehicle.IVehicle`。

Android 13 起默认讲 AIDL；Android 11/12 文档里常见 HIDL `android.hardware.automotive.vehicle@2.0::IVehicle` 的同步 `get/set/subscribe`。本仓库按 **14 的 AIDL** 写。

## 一、它在栈里的位置

```
CarPropertyService / Power 等
        │
   VehicleStub（Java，屏蔽 HIDL/AIDL 差异）
        │ Binder (HwBinder / AIDL)
        ▼
IVehicle（HAL 进程，vendor）
        │
   MCU / 网关 / 车身总线
```

改一款车，通常改 HAL 实现和 `VehiclePropConfig` 表，而不是改 `CarPropertyManager`。

## 二、AIDL 上的 IVehicle（14）

路径：`hardware/interfaces/automotive/vehicle/aidl/`。

能力可以记成四类（方法名以仓库里的 aidl 为准）：

| 能力 | 作用 |
|------|------|
| `getAllPropConfigs` | 上电时告诉 Java 层：我有哪些 prop、类型、area、权限、变化模式 |
| `getValues` | 按请求读（14 上是带 callback 的异步批量读） |
| `setValues` | 写 |
| `subscribe` / `unsubscribe` | 按采样率或 on-change 推送 |

HIDL 2.0 那种「一个 `VehiclePropValue get(...)` 同步返回」不要当成 14 的现状。

## 三、值怎么表示

AIDL 的 `VehiclePropValue` 不再把 `prop` 和所有标量挤在一个 C 结构里那么随便。典型字段：

- `timestamp`：HAL 侧单调时钟，App 用来丢乱序
- `areaId`
- `status`：`AVAILABLE` / `UNAVAILABLE` / `ERROR`
- `value`：`int32Values` / `floatValues` / `int64Values` / `byteValues` / `stringValue`

属性 ID 在请求对象里，不总是嵌在 value 结构中。Java 的 `CarPropertyValue` 是给 App 的包装，和 HAL 结构不是同一个类。

`VehiclePropConfig` 决定：

- 读写（`VehiclePropertyAccess`）
- 变化模式（static / on-change / continuous）
- 采样率范围
- 每个 area 的 min/max
- 读写各要什么权限字符串

权限不是 Java 里写死一张巨大 switch 这么简单，配置可以从 HAL 带上来。

## 四、订阅为什么比轮询重要

连续量（车速）若 App 10ms 一次 `getProperty`，Binder 和 HAL 都会炸。正确路径：

1. Java 服务根据 App 的 `registerCallback(rate)` 合并成对 HAL 的 `SubscribeOptions`
2. HAL 在值变化或达到采样间隔时 `IVehicleCallback` 上报
3. `CarPropertyService` 更新缓存，再回调各 App

vendor 实现如果「订阅了却从不 callback」，上层就会一直看到旧缓存或 UNAVAILABLE。

## 五、vendor 要实现什么

参考实现：`hardware/interfaces/automotive/vehicle/aidl/impl/`（模拟器 fake 车也在这套思路上）。

职责：

1. 提供完整 `PropConfig` 列表（仿真器可以造一份；量产要对实车矩阵）
2. `get/set` 转到你们的车身协议（CAN / SOME-IP / 私有 IPC）
3. 按订阅表推送，注意线程和时钟
4. 正确填 `status`：总线超时是 `UNAVAILABLE` 或 `ERROR`，不要静默填 0

不要在 HAL 里做业务 UI 判断（「行驶中不许调空调」）。那是 UxRestrictions 和策略层的事；HAL 只保证信号真实。

## 六、调试

```bash
# Java 车服务
adb shell dumpsys car_service

# AIDL VHAL 默认实例
adb shell dumpsys android.hardware.automotive.vehicle.IVehicle/default
```

模拟器没有真总线时，用 `car_service` 的 inject 类命令往属性灌值（具体子命令以 `dumpsys car_service --help` / `cmd car_service` 为准，分支之间名字会变）。

## 七、总结

| 要点 | 结论 |
|------|------|
| 契约 | AIDL `IVehicle`（14），不是 App 直连 CAN |
| 配置 | `getAllPropConfigs` 决定能出现哪些 Property |
| 热路径 | subscribe + callback |
| 你改车 | 改 vendor HAL 和属性表 |

---

**下一篇**：[CarPropertyService 属性通路](/articles/car-service/carproperty-source.md)
