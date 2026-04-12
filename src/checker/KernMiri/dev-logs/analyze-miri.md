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


使用示例：

```bash
OSDK_LOCAL_DEV=1 MIRI_LOG=trace MIRI_TRACING=1 cargo osdk miri test
```

* `MIRI_LOG` 控制日志详细级别，默认不输出任何日志
  * 支持 tracing crate 的 [筛选参数]，例如 `MIRI_LOG=miri=trace,rustc_const_eval::interpret=debug`
    对 miri 和 rustc_const_eval 的不同模块设置级别
* `MIRI_TRACING` 控制是否以 JSON 格式输出，设置后将在项目中生成 `trace-时间戳.json` 文件
  * 使用 `find . -name "*.json" -not -path "*/.fingerprint/*"` 命令可查看 JSON 文件路径

JSON 内容示例：

```json
[
{"args":{"name":"rustc"},"name":"thread_name","ph":"M","pid":1,"tid":0},
{".file":"src/helpers.rs",".line":30,"args":{"path":"[\"core\", \"ascii\", \"escape_default\"]"},"cat":"miri::helpers","name":"try_resolve_did","ph":"B","pid":1,"tid":0,"ts":25.112735695853306},
{".file":"src/helpers.rs",".line":30,"args":{"path":"[\"core\", \"ascii\", \"escape_default\"]"},"cat":"miri::helpers","name":"try_resolve_did","ph":"E","pid":1,"tid":0,"ts":67.87289621003023},
{".file":"compiler/rustc_const_eval/src/interpret/eval_context.rs",".line":139,"args":{"layouting":"layout_of","ty":"[u8; 5_usize]"},"cat":"rustc_const_eval::interpret::eval_context","name":"layouting","ph":"B","pid":1,"tid":0,"ts":117.13903337968695},
{".file":"compiler/rustc_const_eval/src/interpret/eval_context.rs",".line":139,"args":{"layouting":"layout_of","ty":"[u8; 5_usize]"},"cat":"rustc_const_eval::interpret::eval_context","name":"layouting","ph":"E","pid":1,"tid":0,"ts":120.90625499346076},
{".file":"src/borrow_tracker/mod.rs",".line":250,"args":{"alloc_size":"Size(5 bytes)","borrow_tracker":"new_allocation","id":"alloc82","kind":"Machine(Machine)"},"cat":"miri::borrow_tracker","name":"borrow_tracker","ph":"B","pid":1,"tid":0,"ts":148.95695174434186},
{".file":"src/borrow_tracker/mod.rs",".line":250,"args":{"alloc_size":"Size(5 bytes)","borrow_tracker":"new_allocation","id":"alloc82","kind":"Machine(Machine)"},"cat":"miri::borrow_tracker","name":"borrow_tracker","ph":"E","pid":1,"tid":0,"ts":156.40194885077625},
{".file":"src/machine.rs",".line":1674,"args":{"data_race":"before_memory_write"},"cat":"miri::machine","name":"data_race","ph":"B","pid":1,"tid":0,"ts":185.52173971591583},
{".file":"src/machine.rs",".line":1674,"args":{"data_race":"before_memory_write"},"cat":"miri::machine","name":"data_race","ph":"E","pid":1,"tid":0,"ts":190.59821664092988},
...
]
```

它可以放到 [perfetto](https://ui.perfetto.dev) 网站查看性能

[perfetto](https://perfetto.dev/docs/)

[tracing](https://perfetto.dev/docs/tracing-101) 概念介绍


![](https://github.com/user-attachments/assets/bb72a224-d600-4606-9588-6b51b8822510)

![](https://github.com/user-attachments/assets/27832eaa-a883-4c47-a4b9-403734413197)

![](https://github.com/user-attachments/assets/fbc65eba-9e96-446e-9811-4f74708291c7)

<details>

<summary>PerfettoSQL Prelude Tables</summary>

我们可以利用 SQL 语法查询 perfetto JSON 内容，而表的结构是固定的：

| 表名             | 用途              | 关键字段                                                  |
| -------------- | --------------- | ----------------------------------------------------- |
| `slice`        | 时间段事件（函数执行、任务等） | `id`, `name`, `ts`, `dur`, `track_id`, `depth`        |
| `thread_track` | 线程时间轨道          | `id`, `utid`                                          |
| `thread`       | 线程信息            | `utid`, `name`, `upid`, `is_main_thread`              |
| `process`      | 进程信息            | `upid`, `pid`, `name`                                 |
| `counter`      | 计数器数据           | `id`, `track_id`, `ts`, `value`                       |
| `args`         | 事件参数            | `arg_set_id`, `flat_key`, `int_value`, `string_value` |

见 <https://perfetto.dev/docs/analysis/sql-tables>。

</details>
