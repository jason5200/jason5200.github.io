# 联发科座舱（MTK）系列导读

> 平行于 [AAOS 中间件主线](/series/aaos.md)：芯片 / BSP 怎么接到 CarService、音频和启动。
>
> 各车企 `vendor/mediatek` 差一截。这里写结构，不抄内部手册。

---

## 路线

```
中间件地图（先知道 CarService / VHAL）
        │
        ▼
MTK 接到哪一层
        │
        ├── 启动 Preloader → LK → Android → WAIT_FOR_VHAL
        ├── AEE：KE / NE / JE / HWT
        └── 音频 HAL / DSP ↔ CarAudio Zone
```

## 目录

| 序号 | 文章 |
|------|------|
| 01 | [BSP 接到 AAOS 的哪一层](/articles/mtk/mtk-aaos-map.md) |
| 02 | [启动：Preloader 到 Android](/articles/mtk/mtk-boot.md) |
| 03 | [AEE：死机和 NE](/articles/mtk/mtk-aee.md) |
| 04 | [音频接到 CarAudioService](/articles/mtk/mtk-audio.md) |

属性读不到，先回 [Vehicle HAL](/articles/permission/vehicle-hal.md)，不要先翻 camera。

仓库：[AAOS-Guide `48-mtk/`](https://github.com/jason5200/AAOS-Guide/tree/main/48-mtk)
