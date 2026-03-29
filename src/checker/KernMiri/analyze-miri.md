# Miri 分析记录

## 调试与日志

### 使用 GDB 调试

一些调试命令和输出被记录在 [#13](https://github.com/KMiri-rs/KMiri/issues/13)。

![](https://github.com/user-attachments/assets/ff1e83b5-a3c4-4932-af51-e18758488333)

![](https://github.com/user-attachments/assets/0d22d09b-6323-4d71-8180-fda93c0397f1)

很少有人尝试用 GDB 调试 Miri。虽然时有人询问如何做，但其作者和维护者通常表示没尝试过这件事。

因此我在 Miri 频道[分享了][managed-to-gdb]这个结果。我用了 1 个多小时学习 GDB 在多进程下如何切换子进程调试，并在手工切换 32 
次 exec catch point，成功进入了 Miri 进程。然后花了两天时间研究怎么让这件事更加自动化，主要是梳理零散的命令和编写 GDB 的 python 插件脚本。

[managed-to-gdb]: https://rust-lang.zulipchat.com/#narrow/channel/269128-miri/topic/Managed.20to.20debug.20miri.20with.20gdb/with/582296758

### 内置的 tracing

[`enter_trace_span`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_const_eval/macro.enter_trace_span.html)

日志阅读 UI：<https://ui.perfetto.dev>

[perfetto](https://perfetto.dev/docs/)

[tracing](https://perfetto.dev/docs/tracing-101) 概念介绍

该功能并不完善，也因为输出太多而很少被使用。
