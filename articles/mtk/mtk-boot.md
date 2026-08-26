# 联发科座舱启动：Preloader 到 Android

> 系列：AAOS-Guide · 48-mtk
> 难度：⭐⭐⭐ 进阶
> 更新：2026-08-26
> 前置知识：[《联发科座舱地图》](/articles/mtk/mtk-aaos-map.md)、[《CarService 启动》](/articles/car-service/carservice-startup-source.md)

---

座舱「慢、黑、卡 Logo」十有八九卡在 **芯片启动链**，不是卡在 `CarPropertyManager`。联发科这条链和手机类似，座舱多了 **多核启动、多屏、等 VHAL**。

文中阶段名是业界常用叫法。具体二进制、串口 log 前缀以你的 BSP 为准。

## 一、一条链记清楚

```
上电 / 复位
  → Preloader（片内 SRAM，初始化 DRAM、选启动介质）
  → LK（Little Kernel：充电、亮 Logo、解 Android 镜像、跳内核）
  → Linux Kernel
  → init / vendor init
  → native 服务、HAL、SurfaceFlinger
  → Zygote / system_server
  → CarServiceHelper bind com.android.car
  → WAIT_FOR_VHAL → ON
```

AAOS 里用户能点 Launcher，往往已经过了 LK。你若只在 App 里优化冷启动，**动不到 Preloader 那 1～2 秒**。

## 二、每一段在干什么

| 阶段 | 失败时你看到的 | 先看什么 |
|------|----------------|----------|
| Preloader | 串口几行就没了、完全黑 | 供电、DDR、eMMC/UFS |
| LK | Logo 停住、进不了动画 | 镜像校验、锁、显示初始化 |
| Kernel | 动画转很久 / 重启 | panic、dts、分区 |
| init | 反复重启 native | `init` 里哪个 service 崩 |
| Framework | 有动画无桌面 | AMS、CarService、VHAL |

座舱比手机更容易卡在 **显示通路**（仪表 + 中控谁先亮）和 **等车辆 HAL**。后者表现为 Android 起来了，属性 `UNAVAILABLE`，电源还在 `WAIT_FOR_VHAL`。

## 三、和 CarService 的衔接

Helper 在 `system_server` 里 `bindServiceAsUser(com.android.car)`。这要求：

1. `com.android.car` 已经装在系统镜像里
2. 进程能起来（VHAL 进程崩溃会拖电源状态机）
3. `IVehicle` default 实例 ready

MTK 启动脚本里常会拉一堆 `vendor.mediatek.*` 服务。其中 **和车无关的**（camera daemon、conn）挂了，不一定挡住 CarService；**和显示、音频、电源 HAL 相关的**挂了，座舱会像没起来。

调试顺序建议：

```bash
adb wait-for-device
adb shell getprop | grep -E "boot|sys.boot"
adb shell dumpsys car_service
adb shell dumpsys android.hardware.automotive.vehicle.IVehicle/default
```

还没 `adbd` 时，只有串口 / 预研口。那是 LK/内核的活，不要在 App log 里找。

## 四、座舱特有的坑

- **多屏**：LK 只亮一块，Android HWC 再接管另一块。两阶段的 timing 会表现为「仪表有、中控无」。
- **AVB / 验签**：量产关了调试口，LK 验签失败就是 Logo 死等。
- **快起**：有的项目 LK 先显示静态仪表，内核后再交权。这是产品策略，不是 CarService 功能。
- **休眠唤醒**：STR 回来要重新过一段时钟和显示 init，和冷启动 log 不一样。对照 [电源一文](/articles/audio/car-power.md) 里的 `SUSPEND_*`。

## 五、你能写进笔记的，和不能写的

能写：阶段顺序、卡在哪一层的判断、和 `WAIT_FOR_VHAL` 的关系。  
不能写：未公开的刷机密钥、内部预研工具的完整命令表、某 OEM 的分区表抄录。

---

**下一篇**：[AEE：死机、NE、JE 怎么下手](/articles/mtk/mtk-aee.md)
