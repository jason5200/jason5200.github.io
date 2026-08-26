# CarPropertyService：鉴权、配置、再下 HAL

> 系列：AAOS-Guide · 01-car-service
> 难度：⭐⭐⭐⭐ 深入
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《CarPropertyManager》](/articles/carservice-api/carproperty-manager.md)、[《Vehicle HAL》](/articles/permission/vehicle-hal.md)

---

App 里一次 `getProperty` / `registerCallback`，真正干活的是 **CarPropertyService**。它不解析 CAN，也不在 Java 里用巨型 `switch (propId)` 写死权限。

下面是结构说明，不是 14 某次提交的逐行摘录。

## 一、中间这一跳

```
CarPropertyManager
    │ Binder（ICarProperty）
    ▼
CarPropertyService
    │ 查 CarPropertyConfig：类型 / area / 读写 / 权限字符串
    ▼
PropertyHalService（HAL 封装）
    │ getValues / setValues / subscribe
    ▼
VehicleStub → IVehicle（AIDL）
    ▼
vendor HAL
```

`VehicleHal.get(VehiclePropValue)` 那种 HIDL 同步接口，不要当成 14 的热路径。

## 二、配置从哪来

服务 init 时向 VHAL 要 **全部属性配置**（`getAllPropConfigs`），做成 `propId → CarPropertyConfig`：

- 数据类型（int / float / string / bytes）
- `areaId` 列表（全局属性通常只有 `0`）
- access：只读 / 只写 / 读写
- change mode：static / on-change / continuous
- 采样率上下限
- **读权限、写权限字符串**（来自 HAL config，不是 Java 一张死表）

所以「模拟器能读车速、量产包读不了」——先 dump 两边的 config，不要先改 App。

## 三、读：先鉴权，再决定打不打 HAL

伪代码（逻辑，不是源码）：

```java
CarPropertyValue getProperty(int propId, int areaId) {
    CarPropertyConfig cfg = configs.get(propId);
    if (cfg == null) throw new IllegalArgumentException("unsupported prop");

    enforcePermission(cfg.getReadPermission()); // 例如 android.car.permission.CAR_SPEED
    assertAreaId(cfg, areaId);
    assertReadable(cfg);

    // 已订阅的连续量往往有缓存；静态量 / 未订阅可能下 HAL
    return propertyHal.get(propId, areaId);
}
```

注意：

- 属性 ID 是 `VehiclePropertyIds.PERF_VEHICLE_SPEED`，没有 `VEHICLE_SPEED` 这种现行常量。
- 客户端 `getProperty(Class<E>, propId, areaId)` 的 `Class` 必须和 config 一致。
- 返回的是 `CarPropertyValue`，带 `status` 和 `timestamp`。缓存命中也要看 status，总线掉了应是 `UNAVAILABLE`，不是假 0。

## 四、写：同样走 config

写比读多两道闸：`isWritable()`、值是否在 min/max（HVAC 温度尤其容易越界被 HAL 拒）。

14 的 HAL 写是批量异步 `setValues` + callback 报错。服务要把 HAL 的错误码转成 App 能理解的 `onErrorEvent` 或异常。不要假设 `setProperty` 返回了就等于风门已经转到位。

## 五、订阅：合并采样率，再跟 HAL 订一份

多个 App 以 1 Hz / 5 Hz / 10 Hz 订同一车速时，服务会 **合并成对 HAL 的一次 subscribe**（取更严的速率），再 fan-out 给各 `CarPropertyEventCallback`。

```
App registerCallback(rate)
  → CarPropertyService 更新订阅表
  → PropertyHalService.subscribe(options)
  → IVehicleCallback / ISubscriptionCallback 上报
  → 更新缓存
  → 回调各 App（注意切线程，不要在 Binder 线程刷 UI）
```

没有人订了，就该 unsubscribe，否则 HAL 会一直采车速。

## 六、权限不是 switch(propId)

旧博客常写成：

```java
switch (propId) {
    case VehiclePropertyIds.VEHICLE_SPEED:
        return Car.PERMISSION_SPEED;
}
```

14 上更接近：每个 `VehiclePropConfig` 带 `readPermission` / `writePermission`。Java 只做 `enforceCallingOrSelfPermission`。vendor 乱填权限字符串，App 就会对不上 Manifest。

控车（门、窗、空调设定）多为 `signature|privileged`。读车速 `CAR_SPEED` 一般是 dangerous，第三方仍可能被车企策略拦。

## 七、和启动、电源的关系

CarPropertyService 依赖 Vehicle 通路已经 `getAllPropConfigs` 成功。电源服务在 `WAIT_FOR_VHAL` 时，属性往往还不可用。调试「开机后前几秒 getProperty 失败」，先看电源状态，再看 HAL。

## 八、总结

| 要点 | 结论 |
|------|------|
| 职责 | 鉴权、校验 config、缓存、合并订阅 |
| 不负责 | 解析 CAN、实现 MCU 协议 |
| 权限 | 来自 HAL 属性配置 |
| 热路径 | subscribe + callback，不是轮询 get |

---

**下一篇**：[权限与驾驶分心](/articles/permission/car-permission.md)
