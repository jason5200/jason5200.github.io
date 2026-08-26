# AEE：联发科座舱死机和 NE 怎么下手

> 系列：AAOS-Guide · 48-mtk
> 难度：⭐⭐⭐ 实战
> 更新：2026-08-26
> 前置知识：[《联发科座舱地图》](/articles/mtk/mtk-aaos-map.md)

---

联发科树上稳定性几乎都收口到 **AEE**（Android Exception Engine / 平台异常收集）。座舱上 JE、NE、KE、卡死，最后往往是一份 db。先认类型，再决定是内核、native 还是 Java。

各 BSP 的落盘路径不一样（`/data/aee_exp`、`/data/vendor/aee_exp` 等），以你机器 `getprop` 和 `ls` 为准。

## 一、常见类型

| 类型 | 含义 | 你先看 |
|------|------|--------|
| KE | Kernel Exception，内核 panic / oops | `ZZ_INTERNAL`、内核栈、oops 原文 |
| NE | Native Exception，native 崩溃 | tombstone、fault addr、`debuggerd` |
| JE | Java Exception，未捕获崩溃 | `system_app_crash`、`system_server` 的 FATAL |
| ANR | 应用无响应 | traces.txt、主线程栈 |
| HWT | Hang Wait Timeout，看门狗类卡死 | 谁没喂狗、当时 CPU/irq |
| SWT | Soft Watchdog，软看门狗 | 多和 system_server / 阻塞有关 |

口头说「车机重启了」，工程师要先问：**是 KE 重启，还是 Java 崩了被拉起来，还是看门狗**。三种 log 完全不同。

## 二、下手顺序

1. **复现一次，留下完整 db**，不要只截 logcat 最后 50 行。
2. 看异常类型和进程名：`system_server`、`surfaceflinger`、`camerahalserver`、`com.android.car` 责任面不同。
3. NE：对着 tombstone 的 `#00 pc` 用 **同一套符号的** 库做 addr2line。版本对不上等于白看。
4. KE：先读 oops 的 `PC is at` / call trace，再怀疑业务。
5. 和车辆无关的崩溃（某相机 daemon）不要先改 CarPropertyService。

`com.android.car` 的 NE：回到中间件，查 VHAL 调用线程、Binder 超时。AEE 只告诉你 **谁崩**，不解释 CAN。

## 三、和 logcat 的关系

AEE db 是「出事那一瞬间的包」。日常还要：

- `logcat -b all`（main / system / crash / events）
- 内核 `dmesg` / `pstore`
- 厂商 mobile log（有的项目叫 mobile_log / debuglogger）

只开 logcat 抓不到 KE；只留 AEE 又看不见崩溃前 10 分钟的车速订阅。两边都要。

## 四、座舱上特别烦的几类

- **熄火瞬间 NE**：电源回调里同步等 HAL，对照 [电源](/articles/audio/car-power.md)，别在 `SHUTDOWN_PREPARE` 里死锁。
- **多屏 HWC hang**：HWT，看起来像整机卡死，其实显示线程没回来。
- **相机 pipeline NE**：倒车进 R 档才崩，和 VHAL 档位上报是两条线，要一起看时序。

## 五、原则

不把内部解码脚本全文贴到公网。公网文写到：**类型、先看哪份文件、如何对齐符号**。你们车间的一键抓取脚本留在内网。

---

**下一篇**：[音频：MTK HAL 接到 CarAudioService](/articles/mtk/mtk-audio.md)
