# 文章归档

> 按系列归档，共 **118 篇**。车载主线 29 篇，Framework 48 篇，AI（含选读）41 篇。

---

## 车载 Android（AAOS）· 29 篇

| 文章 |
|------|
| [车载 Android 全景：AAOS 到底是什么](articles/00-overview/aaos-intro.md) |
| [车载中间件地图](articles/00-overview/middleware.md) |
| [CarService 架构](articles/car-service/carservice-architecture.md) |
| [CarService 启动流程源码](articles/car-service/carservice-startup-source.md) |
| [CarPropertyManager：读写车辆属性](articles/carservice-api/carproperty-manager.md) |
| [CarPropertyService 属性读写源码](articles/car-service/carproperty-source.md) |
| [电源状态管理](articles/audio/car-power.md) |
| [空调控制](articles/audio/car-hvac.md) |
| [车辆传感器：迁到 CarPropertyManager](articles/audio/car-sensor.md) |
| [车辆静态信息](articles/audio/car-info.md) |
| [驾驶分心 UX Restrictions](articles/audio/car-ux.md) |
| [CarService 权限模型](articles/permission/car-permission.md) |
| [Vehicle HAL](articles/permission/vehicle-hal.md) |
| [多屏：Display 到 Surface](articles/multi-display/multi-display.md) |
| [Cluster 仪表盘](articles/multi-display/cluster.md) |
| [HUD](articles/multi-display/hud.md) |
| [倒车影像与环视](articles/perf/reverse-camera.md) |
| [CarAudioService](articles/audio/car-audio-service.md) |
| [多媒体：音频焦点与分区](articles/permission/multimedia.md) |
| [蓝牙电话 HFP](articles/audio/bluetooth-hfp.md) |
| [导航](articles/ota/navigation.md) |
| [OTA 与 A/B 分区](articles/ota/ota-upgrade.md) |
| [启动到座舱可用](articles/perf/boot-process.md) |
| [冷启动优化](articles/perf/cold-start.md) |
| [低内存优化](articles/perf/memory-optimization.md) |
| [V2X（概述）](articles/ota/v2x.md) |
| [功能安全 ISO 26262（概述）](articles/ota/functional-safety.md) |
| [CAN 总线安全（概述）](articles/ota/can-security.md) |
| [模拟器与 HIL](articles/perf/testing.md) |

导读：[series/aaos.md](series/aaos.md)

---

## Android Framework · 48 篇

| 文章 |
|------|
| [Binder 机制总览](articles/framework/binder-overview.md) |
| [一次 Binder 通信全流程](articles/framework/binder-full-flow.md) |
| [Binder 驱动层](articles/framework/binder-driver.md) |
| [Binder mmap 一次拷贝](articles/framework/binder-mmap-deep.md) |
| [binder_transaction](articles/framework/binder-transaction-source.md) |
| [binder_proc / binder_thread](articles/framework/binder-struct-source.md) |
| [Binder 连接池](articles/framework/binder-pool.md) |
| [Binder 线程池](articles/framework/binder-threadpool-source.md) |
| [AIDL：in/out/inout](articles/framework/aidl-deep.md) |
| [AIDL 生成代码](articles/framework/aidl-generated.md) |
| [oneway](articles/framework/oneway.md) |
| [Handler 消息机制](articles/framework/handler-message-mechanism.md) |
| [Handler native 唤醒](articles/framework/handler-native-wakeup.md) |
| [Looper C++](articles/framework/looper-cpp.md) |
| [Looper epoll](articles/framework/looper-epoll-source.md) |
| [MessageQueue native](articles/framework/messagequeue-source.md) |
| [IdleHandler](articles/framework/idlehandler.md) |
| [同步屏障](articles/framework/sync-barrier.md) |
| [HandlerThread](articles/framework/handlerthread.md) |
| [Looper 退出](articles/framework/looper-exit.md) |
| [BlockCanary](articles/framework/blockcanary.md) |
| [AMS 启动流程](articles/framework/ams-startup.md) |
| [AMS 进程管理](articles/framework/ams-process-source.md) |
| [Activity 启动模式](articles/framework/launch-mode.md) |
| [生命周期与异常恢复](articles/framework/lifecycle-restore.md) |
| [Service 启动与绑定](articles/framework/service-bind.md) |
| [Service 源码](articles/framework/service-source.md) |
| [BroadcastReceiver](articles/framework/broadcast.md) |
| [ContentProvider](articles/framework/contentprovider.md) |
| [ContentProvider 跨进程](articles/framework/contentprovider-source.md) |
| [Window / WindowManager](articles/framework/window.md) |
| [WMS](articles/framework/wms-window.md) |
| [Choreographer](articles/framework/choreographer.md) |
| [View 事件分发](articles/framework/touch-event.md) |
| [事件分发源码](articles/framework/touch-source.md) |
| [measure / layout / draw](articles/framework/draw-process.md) |
| [measure 源码](articles/framework/measure-source.md) |
| [layout / draw 源码](articles/framework/layout-draw-source.md) |
| [ViewRootImpl](articles/framework/viewrootimpl-source.md) |
| [硬件加速](articles/framework/hw-render-source.md) |
| [invalidate / requestLayout](articles/framework/invalidate-requestlayout.md) |
| [滑动冲突](articles/framework/scroll-conflict.md) |
| [RecyclerView 缓存](articles/framework/recyclerview.md) |
| [RecyclerView 源码](articles/framework/recyclerview-source.md) |
| [ClassLoader](articles/framework/classloader.md) |
| [内存泄漏](articles/framework/memory-leak.md) |
| [ANR](articles/framework/anr.md) |
| [热修复](articles/framework/hotfix.md) |

导读：[series/framework.md](series/framework.md)

---

## AI 上车 · 41 篇（含选读）

主线 7 篇见 [series/ai.md](series/ai.md)。下列按侧边栏顺序。

| 文章 |
|------|
| [Transformer 原理](articles/ai/transformer.md) |
| [注意力数学推导](articles/ai/attention-math.md) |
| [Llama 注意力源码](articles/ai/llama-attention.md) |
| [llama.cpp KV Cache](articles/ai/llamacpp-kvcache.md) |
| [llama.cpp RoPE](articles/ai/llamacpp-rope.md) |
| [llama.cpp 采样](articles/ai/llamacpp-sampling.md) |
| [预训练到微调](articles/ai/pretrain-finetune.md) |
| [模型量化](articles/ai/quantization.md) |
| [量化误差](articles/ai/quantization-error.md) |
| [QAT](articles/ai/qat.md) |
| [蒸馏与剪枝](articles/ai/distill-prune.md) |
| [端侧推理框架对比](articles/ai/inference-framework.md) |
| [端侧推理性能优化](articles/ai/inference-perf.md) |
| [MediaPipe LLM](articles/ai/mediapipe.md) |
| [ONNX Runtime Mobile](articles/ai/onnx.md) |
| [Embedding](articles/ai/embedding.md) |
| [Embedding 源码](articles/ai/embedding-source.md) |
| [向量数据库](articles/ai/vectordb.md) |
| [端侧推理可行方案](articles/ai/on-device-llm.md) |
| [车载语音助手](articles/ai/voice-assistant.md) |
| [座舱 Agent](articles/ai/agent-cockpit.md) |
| [ReAct](articles/ai/react.md) |
| [多 Agent](articles/ai/multi-agent.md) |
| [Agent 记忆](articles/ai/agent-memory.md) |
| [LangChain / LlamaIndex](articles/ai/langchain.md) |
| [车载 Agent 安全](articles/ai/agent-security.md) |
| [Agent 可观测性](articles/ai/agent-observability.md) |
| [LoRA](articles/ai/lora.md) |
| [端侧多模态部署](articles/ai/multimodal-deploy.md) |
| [TTS 端侧化](articles/ai/tts.md) |
| [Prompt](articles/ai/prompt.md) |
| [大模型安全](articles/ai/llm-safety.md) |
| [工程化实践](articles/ai/ai-engineering.md) |
| [车载 RAG](articles/rag/car-rag.md) |
| [RAG 向量检索](articles/rag/rag-search-source.md) |
| [SQLite 向量](articles/rag/sqlite-vector.md) |
| [RAG 分块](articles/ai/rag-chunking.md) |
| [混合检索](articles/ai/rag-hybrid.md) |
| [RAG 评估](articles/ai/rag-eval.md) |
| [车载多模态](articles/multimodal/multimodal.md) |
| [Function Calling](articles/agent/agent-framework.md) |

---

欢迎通过 [GitHub](https://github.com/jason5200) 纠错。源码对照 AOSP `android-14.0.0_r67`。
