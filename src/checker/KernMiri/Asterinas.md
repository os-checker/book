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

## 运行测试

全部测试：
* `make test` 运行用户态测试
* `make ktest` 内核测试

指定测试：
* `./tools/run_ktest.sh --crates ostd -- max_segment_creation`

```make
.PHONY: ktest2
ktest2: CONSOLE = ttyS0
ktest2: initramfs $(CARGO_OSDK)
	./tools/run_ktest.sh \
        --crates ostd \
        -- max_segment_creation $(CARGO_OSDK_TEST_ARGS)
```

## Qemu 加载内核

/<details>

<summary>Qemu 加载内核的方式和所需的文件格式</summary>

1. 直接加载内核 (使用 `-kernel` 参数)
如果你想绕过启动加载器（如 GRUB），直接让 QEMU 把内核读入内存并启动，通常需要以下格式：

*   **Linux 内核常用格式：**
    *   **x86/x86_64:** 通常使用 **`bzImage`** 或 **`vmlinuz`**。这是 Linux 编译出来的标准压缩内核格式。
    *   **ARM/ARM64:** 常用 **`zImage`** (压缩)、**`Image`** (未压缩的原始二进制) 或 **`uImage`** (带 U-Boot 头的格式)。
*   **ELF 格式 (非常常用):**
    *   **支持情况：** QEMU 原生支持加载 **ELF** 格式的内核。
    *   **适用场景：** 它是**裸机开发（Bare Metal）**、操作系统实验（如 MIT 6.828）或嵌入式开发的首选。

    *   **优势：** ELF 文件包含符号信息和段地址，QEMU 能自动识别该把代码加载到内存的哪个位置（入口点）。
    *   **特定协议：** 如果是 x86 裸机内核，通常需要符合 **Multiboot** 规范（在 ELF 中包含 Multiboot Header），QEMU 才能正确引导。
*   **Raw Binary (原始二进制 `.bin`):**
    *   通常用于极简的裸机程序。QEMU 会根据不同架构将其加载到默认地址（如 ARM 某些单板加载到 `0x10000`）。

2. 通过镜像启动 (使用 ISO/磁盘镜像)
**ISO 文件不是内核格式**，它是一个完整的“光盘镜像”。

*   **如果你只有 ISO：** 你不能用 `-kernel` 加载它。你应该使用 `-cdrom myos.iso` 或 `-drive file=myos.iso,format=raw`。
*   **内部逻辑：** QEMU 会启动 BIOS 或 UEFI 固件，固件再去 ISO 里找 Bootloader（如 GRUB 或 ISOLINUX），由 Bootloader 最终去加载里面的内核。

3. 核心差异对比

| 特性 | `-kernel <file>` | `-cdrom/hda <image>` |
| :--- | :--- | :--- |
| **支持格式** | **ELF**, `bzImage`, `zImage`, `uImage`, `.bin` | **ISO**, `qcow2`, `img`, `vmdk` |
| **启动流程** | **直接加载到内存**，跳过 Bootloader | 模拟硬件启动，需经过 **BIOS/UEFI** |
| **额外要求** | 通常需要配合 `-initrd` (文件系统) 和 `-append` (启动参数) | 内核和文件系统都已封装在镜像内 |
| **常见用途** | 内核开发、裸机实验、快速测试 | 运行完整的操作系统（如 Ubuntu, Windows） |

</details>

星绽使用 `boot.method` 指定生成何种产物，默认为 `QemuDirect` (Qemu 的 --kernel 方式)，但可通过 osdk 参数或者 toml 
配置文件指定（比如 `grub-rescue-iso`）。

## osdk

* 通过本地 make 脚本安装的 osdk，会在 `cargo osdk new` 期间，将 ostd 依赖指向本地路径。这是通过环境变量 `OSDK_LOCAL_DEV` 控制的。

```bash
# 终端 1
cargo osdk test --qemu-args='-s -S'

# 终端 2
## 查看 elf 格式，32 位
file target/osdk/myos-osdk-bin.qemu_elf
#target/osdk/myos-osdk-bin.qemu_elf: ELF 64-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, with debug_info, not stripped
gdb -ex "set architecture i386:x86-64" -ex "target remote :1234" target/osdk/myos-osdk-bin.qemu_elf
```

`ENABLE_KVM=0 LOG_LEVEL=info make ktest`

```rust
#[cfg(ktest)]
mod tests {
    use ostd::prelude::*;

    #[ktest]
    #[should_panic]
    fn max_segment_creation() {
        // Upstream FrameAllocator panics when attempting to allocate a segment with usize::MAX frames
        let max_frames = usize::MAX;
        let _ = ostd::mm::FrameAllocOptions::new().alloc_segment(max_frames);
    }
}
```

b ostd::mm::frame::allocator::FrameAllocOptions::new

b myos::__ostd_main
b ostd::panic::__ostd_panic_handler

cargo osdk run --gdb-server wait-client,addr=:1234
cargo osdk debug --remote :1234

```
(gdb) info variables IN_PANIC
All variables matching regular expression "IN_PANIC":
File src/cpu/local/cell.rs:
48:     static ostd::cpu::local::cell::CpuLocalCell<bool>;
Non-debugging symbols:
0xffffffff882f75c1  ostd::panic::__ostd_panic_handler::IN_PANIC

# 调试 thread locals
# 尝试将字面量视为指针
p *(0xffffffff882f75c1 as *const u8)
# 或者复用这个地址
set $_is_panic = 0xffffffff882f75c1
p *($_is_panic as *const u8)
```


## 词汇表

CPU:
* BSP (BootStrap Processor)：引导处理器。计算机上电启动时唯一主动运行的处理器。它先完成 BIOS 
  初始化、加载操作系统内核等核心引导工作，之后可唤醒所有 AP，还能协调管理包括 AP 在内的所有处理器，其编号通常固定为0。
* AP (Application Processor)：应用处理器。除引导处理器外的其他处理器，上电后处于等待状态，需等待
  BSP 发送唤醒信号才启动。唤醒后会执行简化版引导程序，后续主要承接 BSP 分配的应用计算、任务处理等工作，分担系统负载，提升多任务处理效率。

