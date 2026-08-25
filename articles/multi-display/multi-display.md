# 车机多屏显示：从 Display 到 Surface 的链路

> 系列：AAOS-Guide · 03-multi-display
> 难度：⭐⭐⭐ 进阶
> 更新：2026-08-18
> 前置知识：《CarService 架构》、Android Surface/Window 基础

---

## 一、为什么车机是多屏

智能座舱和手机最大的交互差异之一，就是**多屏**：

| 屏幕 | 作用 | 特点 |
|------|------|------|
| 仪表盘（Cluster） | 车速、导航、警示 | 驾驶员专属，信息密度低、防分心 |
| 中控屏（Center） | 导航、娱乐、设置 | 主交互屏，功能最丰富 |
| 副驾屏（Passenger） | 影音娱乐 | 面向乘客，可放视频 |
| HUD / 后排屏 | 增强信息 | 附加屏 |

这些屏幕**由同一个 Android 系统驱动**，但显示内容、分辨率、方向可能完全不同。这就是 `Multi-Display` 要解决的问题。

## 二、核心概念：Display 与 Surface

理解多屏，先分清两个关键对象：

| 概念 | 是什么 | 类比 |
|------|--------|------|
| **Display** | 一块物理/逻辑屏幕的抽象 | 一块「画布」 |
| **Surface** | 一块可绘制的缓冲区的抽象 | 画布上的「图层」 |

**关系**：一个 Display 上可以叠加多个 Surface；一个 App 的每个 Window 对应一块 Surface。

```
┌───────────── Display（中控屏）─────────────┐
│  Surface A（Launcher）                     │
│  Surface B（地图 App）                     │
│  Surface C（状态栏）                       │
└────────────────────────────────────────────┘
```

## 三、显示链路：从 App 到屏幕

一次画面显示，数据要经过这条链路：

```
App 绘制（View 树）
   │
   ▼
Window（每个 Activity 一个）
   │
   ▼
Surface（缓冲区，App 往这里画）
   │
   ▼
SurfaceFlinger（系统合成器）
   │
   ▼
Hardware Composer（HWC，硬件合成）
   │
   ▼
物理屏幕（Display）
```

**关键理解**：
- App 只负责往自己的 **Surface** 上画，不关心最终显示在哪块屏。
- **SurfaceFlinger** 负责把所有 Surface 合成，再按 Display 配置输出。
- **HWC** 能做硬件合成时，部分工作下沉到 GPU/显示硬件，省电省带宽。

## 四、车机多屏的特殊之处

### 1. Display 由系统动态管理

车机的屏幕数量、分辨率在开机时由系统配置（`DisplayManagerService` 管理）。App 需要**查询**有哪些 Display，而不是写死。

```kotlin
val displayManager = getSystemService(DisplayManager::class.java)
val displays = displayManager.displays  // 所有屏幕

displays.forEach { display ->
    Log.d(TAG, "Display ${display.displayId}: " +
        "${display.name}, ${display.width}x${display.height}")
}
```

### 2. 指定在哪个屏显示

App 可以用 `Presentation` 类或 `ActivityOptions.setLaunchDisplayId()` 把内容放到指定屏幕：

```kotlin
// 方式一：把 Activity 启动到指定屏幕
val options = ActivityOptions.makeBasic()
options.launchDisplayId = clusterDisplayId   // 仪表盘屏
startActivity(intent, options.toBundle())

// 方式二：用 Presentation 在副屏显示自定义 UI
class ClusterPresentation(context: Context, display: Display) :
    Presentation(context, display) {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.cluster_view)
    }
}
```

### 3. 不同屏有不同交互约束

- 仪表盘屏：**禁止**文字输入、视频、复杂交互（驾驶分心限制）。
- 中控屏：交互最完整。
- 副驾屏：可播放视频，但音频路由有讲究。

系统通过 `CarUxRestrictionsManager` 下发这些限制，App 要据此适配 UI。

## 五、一个多屏 Launcher 的架构示意

```
Car Launcher App
│
├── CenterDisplayActivity（中控屏）
│     └── 应用网格、导航卡片
│
├── ClusterPresentation（仪表盘屏）
│     └── 只显示车速、导航箭头（只读，无交互）
│
└── DisplayListener
      └── 监听屏幕热插拔（如后排屏插入）
```

**要点**：
- 用 `DisplayManager.registerDisplayListener` 监听屏幕增删。
- 仪表盘内容用 `Presentation` 而非完整 Activity，轻量且可控。
- 中控屏与仪表盘屏之间通过进程内事件或 Binder 同步数据。

## 六、常见坑与最佳实践

| 坑 | 建议 |
|----|------|
| 写死 displayId | 用 DisplayManager 动态查询 |
| 屏幕热插拔未处理 | 注册 DisplayListener |
| 仪表盘屏放交互控件 | 遵守驾驶分心限制，只读展示 |
| 多屏绘制性能差 | 控制刷新率，副屏内容按需更新 |
| 分辨率适配写死 dp | 多屏分辨率不同，用自适应布局 |

## 七、总结

| 要点 | 结论 |
|------|------|
| 车机为何多屏 | 仪表盘/中控/副驾分工不同 |
| Display vs Surface | 屏幕抽象 vs 缓冲区抽象 |
| 显示链路 | App → Window → Surface → SurfaceFlinger → HWC → 屏 |
| 多屏关键 API | DisplayManager、Presentation、ActivityOptions |
| 特殊约束 | 驾驶分心限制、屏幕热插拔 |

---

**下一篇预告**：《车机冷启动优化实战》

> 配套仓库：[AAOS-Guide](https://github.com/jason5200/AAOS-Guide) · 实战 Demo：[Car-Launcher-Demo](https://github.com/jason5200/Car-Launcher-Demo)
