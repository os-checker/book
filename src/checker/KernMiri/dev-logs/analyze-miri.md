# Miri 分析记录

## 内存类型

## 内存分配

### InterpCx::allocate

[`InterpCx::allocate`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_const_eval/interpret/struct.InterpCx.html#method.allocate)

* 对 `Sized` 和非 `extern` 类型分配
* AllocId 自增作为新的 id
* 采用解释器的默认分配器：Global（正常模式）或者 Isolated（native-lib 模式）
* 根据 [`PANIC_ON_ALLOC_FAIL`] 处理分配失败（panic 或者返回 Err）
* 分配字节（得到实际分配指针）并清零初始化 [`MiriAllocBytes::zeroed`](https://doc.rust-lang.org/nightly/nightly-rustc/miri/struct.MiriAllocBytes.html#method.zeroed)
* 将分配指针信息插入到 Miri 分配状态 [`InterpCx::insert_allocation`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_const_eval/interpret/struct.InterpCx.html#method.insert_allocation)
  * 确保分配的大小不超过 isize
  * 确保当前分配类型为 MemoryKind::Machine
  * 创建一个新 AllocId（尚未关联分配）
  * Miri 初始化本地分配状态 [`MiriMachine::init_local_allocation`](https://doc.rust-lang.org/nightly/nightly-rustc/miri/struct.MiriMachine.html#method.init_local_allocation)
    * 确保不是 MiriMemoryKind::Global 分配类型
    * 更新 borrow tracker 状态
      * 自增 BorTag，并在 `root_ptr_tags` 关联 AllocId 和 BorTag
      * stack borrow：对 MemoryKind::Stack 分配赋予 Unique 权限；对其他分配赋予 Shared 权限
      * tree borrow
    * 创建 data_race 检测相关的分配状态
  * 添加上面额外的分配信息 [`miri::AllocExtra`](https://doc.rust-lang.org/nightly/nightly-rustc/miri/struct.AllocExtra.html)
  * 将 AllocId、分配类型和情况插入 [`MonoHashMap<AllocId, (MemoryKind<MiriMemoryKind>, Allocation`](https://doc.rust-lang.org/nightly/nightly-rustc/miri/machine/struct.MiriMachine.html#associatedtype.MemoryMap)
* 调整分配指针 [`adjust_alloc_root_pointer`](https://doc.rust-lang.org/nightly/nightly-rustc/miri/alloc_addresses/trait.EvalContextExt.html#method.adjust_alloc_root_pointer)
  * 将 [`Pointer<CtfeProvenance>`] 的默认参数 CtfeProvenance 转换为 [`miri::Provenance`](https://doc.rust-lang.org/nightly/nightly-rustc/miri/enum.Provenance.html)
  * addr_from_alloc_id
* ptr_with_meta_to_mplace

[`PANIC_ON_ALLOC_FAIL`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_const_eval/interpret/trait.Machine.html#associatedconstant.PANIC_ON_ALLOC_FAIL
[`Pointer<CtfeProvenance>`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/interpret/struct.Pointer.html

### addr_from_alloc_id_uncached

```rust
fn addr_from_alloc_id_uncached(
    &self,
    global_state: &mut GlobalStateInner,
    alloc_id: AllocId,
    memory_kind: MemoryKind,
) -> InterpResult<'tcx, u64> {
```

在分配立即发生之后、调整指针期间对分配进行进一步处理，最终返回一个地址，该地址的含义取决于分配类型。

* GenMC 并发检测；尚未实现分配处理逻辑
* 在 `-Zmiri-native-lib` 模式下（对于 FFI 指针），Miri 有多种策略处理
  * 全局分配将复制到 mmap 创建的单独的内存中 (Isolated 分配器)
  * 非全局分配使用原来的指针地址
  * 函数和虚表使用一个假的地址充当唯一指针
* 在正常模式下
  * 栈分配
  * 堆分配

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
