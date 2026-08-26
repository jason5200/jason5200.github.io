# CarPowerManagementService：上电、熄火准备、挂起

> 系列：AAOS-Guide · 01-car-service
> 难度：⭐⭐⭐⭐ 深入
> 更新：2026-08-26
> 对照：[AOSP android-14.0.0_r67](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《CarService 启动流程》](/articles/car-service/carservice-startup-source.md)

---

车的电源不是「亮屏 / 息屏 / 关机」三档。VHAL 上报的 AP 电源状态，由 **CarPowerManagementService（CPMS）** 收成一条状态机，再通知特权监听者。普通应用往往 **没有** 这套 API；能做的是把状态写进自己的 `onPause`。

`CarPowerManager` 在 14 上大量接口是 `@SystemApi`，需要 `android.car.permission` 里与关机流程相关的特权权限。

## 一、为什么必须等 VHAL

开机后 CPMS 先进入 **`WAIT_FOR_VHAL`**：HAL 进程还没 ready，不能假定车速、档位、点火状态是真的。HAL 一旦活着，才迁到 **`ON`**。

这能解释：开机动画结束前 `Car.createCar` 成功了，但 `getProperty` 仍 `UNAVAILABLE`。

## 二、应用会听到的状态（14）

常量在 `CarPowerManager`（数值以 car-lib 为准）：

| 状态 | 含义 |
|------|------|
| `STATE_WAIT_FOR_VHAL` | 等车辆 HAL |
| `STATE_ON` | 座舱可用 |
| `STATE_PRE_SHUTDOWN_PREPARE` | 关机流程已请求，显示/音频可能仍在 |
| `STATE_SHUTDOWN_PREPARE` | 准备熄火；可能跑 Garage Mode（闲时任务） |
| `STATE_SHUTDOWN_ENTER` | 真正进入关机，做最后清理 |
| `STATE_SUSPEND_ENTER` / `STATE_SUSPEND_EXIT` | 挂起（STR）进出 |
| `STATE_HIBERNATION_ENTER` / `EXIT` | 休眠（如果产品开了） |
| `STATE_SHUTDOWN_CANCELLED` | 用户又拧回来了，取消关机 |

网上示意图里的单独 `STATE_SUSPEND`、`STATE_SHUTDOWN` 不要当 14 的 listener 常量抄。

## 三、正确的「我写完了」方式

**没有** `requestShutdownDelay(...)` 这种公开 API。特权监听者用 **带 completion 的 listener**：

```java
powerManager.setListenerWithCompletion(executor, (state, future) -> {
    if (state == CarPowerManager.STATE_SHUTDOWN_PREPARE) {
        persistSession();
        if (future != null) {
            future.complete(null); // 14 上多为 CompletableFuture<Void>
        }
        return;
    }
    if (future != null) {
        future.complete(null);
    }
});
```

要点：

- 不 `complete`，CPMS 会等到超时再往下走（超时可能很长，表现为「熄不了火」）。
- `SHUTDOWN_PREPARE` 上官方文档对 **异步拖延更严**；能同步落盘就同步。允许异步 complete 的状态以你这版 javadoc 为准（常见是 `PRE_SHUTDOWN_PREPARE`、`SUSPEND_ENTER` 等）。
- 更新的 AOSP 把 future 换成了 `CompletablePowerStateChangeFuture`，并带过期时间。**以你编译的 car-lib 为准。**

只有 `CarPowerStateListener`、没有 completion 的，适合「知道一声」；不要承担「必须写完闪存」的职责。

## 四、和 VHAL 的关系

CPMS 会跟 Vehicle HAL 做电源握手（关机推迟、深度睡眠入口等）。Garage Mode 在 `SHUTDOWN_PREPARE` 窗口跑 JobScheduler 类任务（地图更新、日志上传）。产品关了 Garage Mode，这个窗口就短。

挂起（suspend-to-ram）时，Java 世界像暂停；远程唤醒由 SoC / VMCU 负责，不是普通 App 注册一个 Broadcast 就能接收 CAN 唤醒。

## 五、普通座舱 App 怎么做

1. 不要假装能拦关机。
2. 关键数据：`onPause` / `onStop` 就写，不要等电源回调。
3. 若你是系统服务（蓝牙策略、音频），用 `setListenerWithCompletion`，并在每个带 future 的状态 **都 complete**，避免卡死状态机。

## 六、总结

| 要点 | 结论 |
|------|------|
| 先 VHAL 后 ON | `WAIT_FOR_VHAL` |
| 熄火窗口 | `SHUTDOWN_PREPARE` → `SHUTDOWN_ENTER` |
| 完成通知 | completion future，不是 delay API |
| 听众 | 系统 / 特权服务为主 |

---

**下一篇**：[CarAudioService](/articles/audio/car-audio-service.md)
