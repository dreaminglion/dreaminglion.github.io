
## native 崩溃的捕获流程

https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w?

Chromium 的 Breakpad 是目前 Native 崩...

https://chromium.googlesource.com/breakpad/breakpad/+/master

使用 gdb 调试


## 包含 native 崩溃的捕获平台
腾讯的Bugly、阿里的啄木鸟平台、网易云捕、Google 的 Firebase 等等


## 安全模式
保证 app 正常启动 https://mp.weixin.qq.com/s?__biz=MzUxMzcxMzE5Ng==&mid=2247488429&idx=1&sn=448b414a0424d06855359b3eb2ba8569&source=41#wechat_redirect


02

- 崩溃信息获取（崩溃信息、系统信息、内容信息、资源信息、应用信息），分析，复现。

03

- 按照设备年代内存策略，进行不同的内存分配以及动画机制，合理使用设备性能。
- Fresco 使用 native 内存，减少 gc 操作，提高流畅度。

04

- 统一入口，统一监管。

- bitmap 内存重复检测、图片超过屏幕大小。

- oom 监控 Probe https://static001.geekbang.org/con/19/pdf/593bc30c21689.pdf

- gc 监控
```
// 运行的 GC 次数
Debug.getRuntimeStat("art.gc.gc-count");
// GC 使用的总耗时，单位是毫秒
Debug.getRuntimeStat("art.gc.gc-time");
// 阻塞式 GC 的次数
Debug.getRuntimeStat("art.gc.blocking-gc-count");
// 阻塞式 GC 的总耗时
Debug.getRuntimeStat("art.gc.blocking-gc-time");
```


05

Traceview 和 systrace 都是我们比较熟悉的排查卡顿的工具。

Nanoscope 作为基本没有性能损耗的 instruction 工具，非常适合做启动耗时的自动发分析。

systrace是 Android 4.1 新增的性能分析工，我通常使用 systrace 跟踪系统的 I/O 操作、CPU 负载、Surface 渲染、GC 等事件。

如果需要分析 Native 代码的耗时，可以选择 Simpleperf；如果想分析系统调用，可以选择systrace；如果想分析整个程序执行流程的耗时，可以选择 Traceview 或者插桩版本的 systrace.

06

Android Vitals 是 Google Play 官方的性能监控服务，监控有 ANR、启动、帧率三个。

后面我会讲到 Aspect、ASM 和 ReDex 三种插桩技术的实现。插桩获取函数执行的时间。

07
