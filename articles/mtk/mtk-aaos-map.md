# 联发科座舱：BSP 接到 AAOS 的哪一层

> 系列：AAOS-Guide · 48-mtk
> 难度：⭐⭐ 进阶
> 更新：2026-08-26
> 对照：vendor BSP（各车企差一截）；中间件仍按 [AOSP 14](https://github.com/jason5200/AAOS-Guide/blob/main/AOSP_VERSION.md)
> 前置知识：[《中间件地图》](/articles/00-overview/middleware.md)

---

中间件主线讲的是 **Google 这一截**：`android.car` → CarService → `IVehicle`。联发科座舱要多讲一截：**芯片上电之后、属性从哪来、声音从哪出、屏谁合成**。

本系列不替代 Vehicle HAL 文档，只补 **MTK BSP 怎么接到那几根管子上**。文中是座舱上常见的分层，不是某份内部手册的拷贝；命令和目录以你手里的 `vendor/mediatek` 为准。

## 一、先认清两张图

标准 AAOS：

```
App → CarService → IVehicle / Audio / Display
                      │
                 vendor HAL 实现
```

联发科车上，vendor 这一层通常更厚：

```
CarProperty / CarAudio / SurfaceFlinger
        │
   HIDL/AIDL HAL（往往在 vendor 分区）
        │
   MTK 用户态：cam / audio / display / conn / aee
        │
   Kernel 驱动 + DT
        │
   Preloader / LK
        │
   SoC（CPU / GPU / APU / DSP / 显示 / 连接）
```

**CarService 仍然不解析 CAN。** 车身信号还是进你们的 Vehicle HAL；MTK 负责的是 **这块板能不能亮、能不能响、相机能不能出图、Wi-Fi/BT/GNSS 是否起来**。两边在电源、音频、多屏上才会碰头。

## 二、座舱上常见的芯片代际

名字会随项目改，先按「干什么」记，不要背料号：

| 代际（口头） | 座舱上常见用途 | 你更该关心 |
|--------------|----------------|------------|
| MT2712 一代 | 中控 / 仪表较早量产 | 启动、显示、AEE |
| MT8675 / 8676 一带 | 较新 IVI | 多屏、相机、APU、Android 版本 |
| Dimensity Auto 等新平台 | 高算力座舱 | 同一套分层，目录和特性集不同 |

同一料号，车企 BSP 的 `device/`、内核配置、车辆 HAL **都可以完全不同**。所以本系列默认讲 **结构**，点名路径时会写「常见位置」。

## 三、模块怎么切（和中间件对齐）

| 你在车上碰到的问题 | 多半在 MTK 哪一层 | 接到 AAOS 哪边 |
|--------------------|-------------------|----------------|
| 上电黑屏、卡 Logo | Preloader / LK / 内核 / HWC | 启动、多屏 |
| 车速有、媒体没声 | audio HAL / DSP / tinyalsa | [CarAudioService](/articles/audio/car-audio-service.md) |
| 倒车影像花、无图 | imgsensor / mtkcam / 显示通路 | 相机、Surface |
| 属性全 UNAVAILABLE | **不一定是 MTK** | 先 [VHAL](/articles/permission/vehicle-hal.md) |
| 死机、重启、JE | AEE（KE/NE/HWT/SWT） | 稳定性 |
| 蓝牙电话、定位 | conninfra / wmt / GNSS | 连接、导航 |

车辆属性读不到，优先 dump `car_service` 和 `IVehicle`，不要一上来就翻 MTK camera。

## 四、源码在树上的大致位置

公开 AOSP 里几乎没有完整座舱 BSP。量产树常见：

- `vendor/mediatek/` 或车企自己的 `vendor/<oem>/` 里再嵌一套 MTK
- `kernel-*/drivers/` 与 `arch/arm64/boot/dts/`
- `device/mediateksample/` 或 `device/<oem>/<sku>/`：mk、fstab、audio 配置

能对外写的，是 **HAL 接口名、分区角色、log 标签、AEE 类型**。寄存器级、未公开工具链，本系列不写。

## 五、建议阅读顺序

1. 本文：先知道 BSP 不替代 CarService
2. [启动：Preloader 到 Android](/articles/mtk/mtk-boot.md)
3. [AEE：死机和 NE 怎么下手](/articles/mtk/mtk-aee.md)
4. [音频：DSP 接到 CarAudio](/articles/mtk/mtk-audio.md)

电源状态机仍读 [CarPowerManagementService](/articles/audio/car-power.md)。MTK 侧对应的是 **SoC 休眠 / 唤醒 / 时钟**，和 CPMS 握手的是你们的电源 HAL，不是再发明一套 Car API。

---

**下一篇**：[联发科座舱启动：Preloader 到 Android](/articles/mtk/mtk-boot.md)
