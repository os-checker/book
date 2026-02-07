# Asterinas (星绽)

## ktest

ktest 是基于 [ostd] 的内核测试框架，针对 `#![no_std]` 库提供类似 `cargo test` 的体验，由以下一些部分组成：

* [`#[ktest]`] 包装一个测试函数，并生成一个全局静态变量，这个静态变量的类型是 [`KtestItem`]，并放置在 
  `.ktest_array` section（搭配链接脚本工作，比如 `__ktest_array` 和 `__ktest_array_end` 之间的内存区域视为
  `&[KtestItem]`，从而迭代所有测试函数）。
* `cargo osdk test` 基于当前 crate 自动生成一个 `${crate}-test-base` 的 crate，其中设置了全局分配器和测试相关的全局变量。
  所有的测试随系统运行而运行，默认通过 qemu 运行这个测试的操作系统。
* 由于测试的代码嵌入到内核镜像文件中，因此使用 `#[cfg(ktest)]` 控制编译的代码，在正常内核代码中不开启它来排除测试代码，在测试时开启它而编译测试代码。

[ostd-test]: https://docs.rs/ostd
[`KtestItem`]: https://docs.rs/ostd-test/0.17.0/ostd_test/struct.KtestItem.html
[`#[ktest]`]: https://docs.rs/ostd-macros/latest/ostd_macros/attr.ktest.html

编写一个内核测试就像编写一个用户程序的测试一样简单：

```rust
#[cfg(ktest)]
mod tests {
    use ostd::prelude::*;

    #[ktest]
    fn it_works() {
        let memory_regions = &ostd::boot::boot_info().memory_regions;
        assert!(!memory_regions.is_empty());
    }
}
```

<details>

<summary>展开为：</summary>

```rust
#[cfg(ktest)]
mod tests {
    use ostd::prelude::*;
    fn it_works() {
        let memory_regions = &ostd::boot::boot_info().memory_regions;
        if !!memory_regions.is_empty() {
            ::core::panicking::panic("assertion failed: !memory_regions.is_empty()")
        }
    }
    #[used]
    #[unsafe(link_section = ".ktest_array")]
    static it_works_ktest_item_yQQSes82: ostd::ktest::KtestItem = ostd::ktest::KtestItem::new(
        it_works,
        (false, None),
        ostd::ktest::KtestItemInfo {
            module_path: "mylib::tests",
            fn_name: "it_works",
            package: "mylib",
            source: "mylib/src/lib.rs",
            line: 8usize,
            col: 4usize,
        },
    );
}
```

</details>

测试输出：

```bash
$ cd mylib
$ cargo osdk test
[ktest runner] running 194 tests in 5 crates

running 1 tests in crate "mylib"

test mylib::tests::it_works ...it works
 ok

test result: ok. 1 passed; 0 failed; 0 filtered out.

[ktest runner] skipping crate "osdk-frame-allocator".

[ktest runner] skipping crate "osdk-heap-allocator".

[ktest runner] skipping crate "osdk-test-kernel".

[ktest runner] skipping crate "ostd".

[ktest runner] All crates tested.
```

## 词汇表

CPU:
* BSP (BootStrap Processor)：引导处理器。计算机上电启动时唯一主动运行的处理器。它先完成 BIOS 
  初始化、加载操作系统内核等核心引导工作，之后可唤醒所有 AP，还能协调管理包括 AP 在内的所有处理器，其编号通常固定为0。
* AP (Application Processor)：应用处理器。除引导处理器外的其他处理器，上电后处于等待状态，需等待
  BSP 发送唤醒信号才启动。唤醒后会执行简化版引导程序，后续主要承接 BSP 分配的应用计算、任务处理等工作，分担系统负载，提升多任务处理效率。

