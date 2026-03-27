## 测试代码

```bash
cargo osdk new os
```

将 `src/lib.rs` 的测试部分替换为以下内容

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

```bash
# 终端 1 
cargo osdk test --qemu-args="-s -S"
# 终端 2
rust-gdb -x gdb.txt
```

其中 `gdb.txt` 设置了初始化和断点操作：

```
{{#include gdb.txt}}
```


## 调用栈情况

```gdb
>>> bt
#0  unwinding::unwinder::_Unwind_RaiseException (exception=0xffff800260169000) at src/unwinder/mod.rs:155
#1  0xffffffff882f5e23 in unwinding::panicking::begin_panic<unwinding::panic::RustPanic> (exception=...) at src/panicking.rs:24
#2  0xffffffff882e4bd3 in unwinding::panic::begin_panic (payload=...) at src/panic.rs:63
#3  0xffffffff880f06c9 in osdk_test_kernel::panic_handler (info=0xffffdfffffffbd70) at src/lib.rs:73
#4  0xffffffff880f1c2a in osdk_test_kernel::__ostd_panic_handler (info=0xffffdfffffffbd70) at src/lib.rs:55
#5  0xffffffff8804b02a in os_osdk_bin::panic (info=0xffffdfffffffbd70) at src/main.rs:13
#6  0xffffffff883b0406 in core::panicking::panic_fmt (fmt=...) at src/panicking.rs:80
#7  0xffffffff883a9835 in core::panicking::panic_const::panic_const_mul_overflow () at src/panicking.rs:175
#8  0xffffffff882440b8 in ostd::mm::frame::allocator::FrameAllocOptions::alloc_segment_with<(), ostd::mm::frame::allocator::{impl#1}::alloc_segment::{closure_env#0}> (self=0xffffdfffffffbfff, nframes=18446744073709551615, metadata_fn=...) at src/mm/frame/allocator.rs:94
#9  0xffffffff88135eea in ostd::mm::frame::allocator::FrameAllocOptions::alloc_segment (self=0xffffdfffffffbfff, nframes=18446744073709551615)
    at src/mm/frame/allocator.rs:72
#10 0xffffffff881309ec in os::tests::max_segment_creation () at src/lib.rs:12
#11 0xffffffff880e829b in core::ops::function::FnOnce::call_once<fn(), ()> ()
    at /root/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ops/function.rs:250
#12 0xffffffff881244df in unwinding::panicking::catch_unwind::do_call<fn(), ()> (data=0xffffdfffffffc088)
    at /root/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/unwinding-0.2.8/src/panicking.rs:54
#13 0xffffffff881245b8 in __rust_try ()
#14 0xffffffff8812446f in unwinding::panicking::catch_unwind<unwinding::panic::RustPanic, (), fn()> (f=0xffffffff881309c0 <os::tests::max_segment_creation>)
    at /root/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/unwinding-0.2.8/src/panicking.rs:42
#15 0xffffffff8812441a in unwinding::panic::catch_unwind<(), fn()> (f=0xffffffff881309c0 <os::tests::max_segment_creation>)
    at /root/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/unwinding-0.2.8/src/panic.rs:87
#16 0xffffffff882dfe53 in ostd_test::KtestItem::run (self=0xffff800260160200, catch_unwind_impl=0xffffffff883d7670,
    f=0xffffffff880e82b0 <core::ops::function::FnOnce::call_once<osdk_test_kernel::run_crate_ktests::{closure_env#0}, (&ostd_test::PanicInfo)>>)
    at src/lib.rs:155
#17 0xffffffff880f186b in osdk_test_kernel::run_crate_ktests (crate_=0xffff80001314f800, whitelist=0xffffdfffffffccb0) at src/lib.rs:156
#18 0xffffffff880f764c in osdk_test_kernel::run_ktests<core::iter::adapters::map::Map<core::slice::iter::Iter<&str>, osdk_test_kernel::main::{closure#0}::{closure#0}::{closure_env#0}>> (test_whitelist=..., crate_whitelist=...) at src/lib.rs:120
#19 0xffffffff880f79d4 in osdk_test_kernel::main::{closure#0} () at src/lib.rs:43
#20 0xffffffff880e82fe in core::ops::function::FnOnce::call_once<osdk_test_kernel::main::{closure_env#0}, ()> ()
    at /root/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ops/function.rs:250
#21 0xffffffff880e6d6e in core::ops::function::FnOnce::call_once<osdk_test_kernel::main::{closure_env#0}, ()> ()
    at /root/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ops/function.rs:250
#22 0xffffffff881c5938 in alloc::boxed::{impl#31}::call_once<(), (dyn core::ops::function::FnOnce<(), Output=()> + core::marker::Send), alloc::alloc::Global> (
    self=..., args=()) at /root/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/alloc/src/boxed.rs:2206
#23 0xffffffff8821dbf7 in ostd::task::{impl#1}::build::kernel_task_entry () at src/task/mod.rs:188
#24 0xffffffff8825b86a in kernel_task_entry_wrapper ()
```

## catch_unwind

```
─── Output/messages ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
0xffffffff881245c1 in __rust_try ()
─── Assembly ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
>>> disassemble (0xffffffff881245a0 as *const ())
Dump of assembler code for function __rust_try:
   0xffffffff881245a0 <+0>:     sub    rsp,0x18
   0xffffffff881245a4 <+4>:     mov    QWORD PTR [rsp+0x8],rdx
   0xffffffff881245a9 <+9>:     mov    QWORD PTR [rsp+0x10],rsi
   0xffffffff881245ae <+14>:    mov    rax,rdi
   0xffffffff881245b1 <+17>:    mov    rdi,QWORD PTR [rsp+0x10]
   0xffffffff881245b6 <+22>:    call   rax
   0xffffffff881245b8 <+24>:    jmp    0xffffffff881245ba <__rust_try+26>
   0xffffffff881245ba <+26>:    xor    eax,eax
   0xffffffff881245bc <+28>:    add    rsp,0x18
   0xffffffff881245c0 <+32>:    ret
   0xffffffff881245c1 <+33>:    mov    rdi,QWORD PTR [rsp+0x10]
   0xffffffff881245c6 <+38>:    mov    rsi,rax
   0xffffffff881245c9 <+41>:    mov    rax,QWORD PTR [rsp+0x8]
   0xffffffff881245ce <+46>:    call   rax
   0xffffffff881245d0 <+48>:    mov    eax,0x1
   0xffffffff881245d5 <+53>:    add    rsp,0x18
   0xffffffff881245d9 <+57>:    ret
─── Stack ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[0] from 0xffffffff881245c1 in __rust_try
[1] from 0xffffffff8812446f in unwinding::panicking::catch_unwind<unwinding::panic::RustPanic, (), fn()>+63 at /root/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/unwinding-0.2.8/src/panicking.rs:42
[2] from 0xffffffff8812441a in unwinding::panic::catch_unwind<(), fn()>+26 at /root/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/unwinding-0.2.8/src/panic.rs:87
[3] from 0xffffffff882dfe53 in ostd_test::KtestItem::run+67 at src/lib.rs:155
[4] from 0xffffffff880f186b in osdk_test_kernel::run_crate_ktests+4283 at src/lib.rs:156
[5] from 0xffffffff880f764c in osdk_test_kernel::run_ktests<core::iter::adapters::map::Map<core::slice::iter::Iter<&str>, osdk_test_kernel::main::{closure#0}::{closure#0}::{closure_env#0}>>+860 at src/lib.rs:120
[6] from 0xffffffff880f79d4 in osdk_test_kernel::main::{closure#0}+68 at src/lib.rs:43
[7] from 0xffffffff880e82fe in core::ops::function::FnOnce::call_once<osdk_test_kernel::main::{closure_env#0}, ()>+14 at /root/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ops/function.rs:250
[8] from 0xffffffff880e6d6e in core::ops::function::FnOnce::call_once<osdk_test_kernel::main::{closure_env#0}, ()>+14 at /root/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ops/function.rs:250
[9] from 0xffffffff881c5938 in alloc::boxed::{impl#31}::call_once<(), (dyn core::ops::function::FnOnce<(), Output=()> + core::marker::Send), alloc::alloc::Global>+40 at /root/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/alloc/src/boxed.rs:2206
```

[`unwinding::panicking::catch_unwind`][catch_unwind] 调用了 intrinsic [`catch_unwind`][i-catch-unwind]，并提供 `do_call` 和 `do_catch` 
这两个处理函数以及一个数据指针。编译器在代码生成期间会把 intrinsic `catch_unwind` 内联成一段 [`__rust_try`] 代码，并在传入的函数（闭包）生成一段
landing pad 指令，以供 eh_personality 回溯到这个函数的时候执行捕获 `do_catch` 并恢复正常的执行流。

> landing pad: A section of user code intended to catch, or otherwise clean up
> after, an exception. It gains control from the exception runtime via the
> personality routine, and after doing the appropriate processing either merges
> into the normal user code or returns to the runtime by resuming or raising a
> new exception.
>
> src: [Itanium C++ ABI: Exception Handling](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html)

```rust
// std::intrinsics::catch_unwind
pub unsafe fn catch_unwind(
    _try_fn: fn(*mut u8),
    _data: *mut u8,
    _catch_fn: fn(*mut u8, *mut u8),
) -> i32;
```

[catch_unwind]: https://github.com/nbdd0121/unwinding/blob/a01c58e7e47a11ba158275e3d2fe04d18f9755a6/src/panicking.rs#L28
[i-catch-unwind]: https://doc.rust-lang.org/stable/std/intrinsics/fn.c
[`__rust_try`]: https://github.com/rust-lang/rust/blob/80d0e4be6f15899649ba31669077c59a986f96cc/compiler/rustc_codegen_llvm/src/intrinsic.rs#L839

## 调试过程

```
>>> x/i frame.fde_result.fde.initial_address
   0xffffffff880f0250 <osdk_test_kernel::panic_handler>:        sub    rsp,0x238
>>> info symbol frame.fde_result.fde.initial_address
osdk_test_kernel::panic_handler in section .text
```

```
>>> x/i frame.fde_result.fde.initial_address
   0xffffffff8804b020 <os_osdk_bin::panic>:     push   rax
>>> info symbol frame.fde_result.fde.initial_address
__rustc::rust_begin_unwind in section .text
>>> disassemble /s frame.fde_result.fde.initial_address
Dump of assembler code for function os_osdk_bin::panic:
src/main.rs:
9       fn panic(info: &core::panic::PanicInfo) -> ! {
   0xffffffff8804b020 <+0>:     push   rax
   0xffffffff8804b021 <+1>:     mov    QWORD PTR [rsp],rdi

10          unsafe extern "Rust" {
11              pub fn __ostd_panic_handler(info: &core::panic::PanicInfo) -> !;
12          }
13          unsafe { __ostd_panic_handler(info); }
   0xffffffff8804b025 <+5>:     call   0xffffffff880f1c20 <osdk_test_kernel::__ostd_panic_handler>
```


/<details>

<summary>`cargo osdk new os` 生成的测试入口</summary>

```rust
// target/osdk/os-test-base/src/main.rs (os_osdk_bin)
#![no_std]
#![no_main]

#![feature(linkage)]

extern crate os;

#[panic_handler]
fn panic(info: &core::panic::PanicInfo) -> ! {
    unsafe extern "Rust" {
        pub fn __ostd_panic_handler(info: &core::panic::PanicInfo) -> !;
    }
    unsafe { __ostd_panic_handler(info); }
}

mod default_frame_allocator {
    use ostd::mm::frame::GlobalFrameAllocator;

    use osdk_frame_allocator::FrameAllocator;
    static FRAME_ALLOCATOR: FrameAllocator = FrameAllocator;

    // SAFETY: The name does not collide with other symbols.
    #[unsafe(no_mangle)]
    #[linkage = "weak"]
    static __GLOBAL_FRAME_ALLOCATOR_REF: &'static dyn GlobalFrameAllocator = &FRAME_ALLOCATOR;
}

mod default_heap_allocator {
    use ostd::mm::heap::GlobalHeapAllocator;

    use osdk_heap_allocator::{HeapAllocator, type_from_layout};
    static HEAP_ALLOCATOR: HeapAllocator = HeapAllocator;

    // SAFETY: The name does not collide with other symbols.
    #[unsafe(no_mangle)]
    #[linkage = "weak"]
    static __GLOBAL_HEAP_ALLOCATOR_REF: &'static dyn GlobalHeapAllocator = &HEAP_ALLOCATOR;

    // SAFETY: The name does not collide with other symbols.
    #[unsafe(no_mangle)]
    #[linkage = "weak"]
    fn __global_heap_slot_info_from_layout(
        layout: core::alloc::Layout,
    ) -> Option<ostd::mm::heap::SlotInfo> {
        type_from_layout(layout)
    }
}


extern crate osdk_test_kernel;

// SAFETY: The name does not collide with other symbols.
#[unsafe(no_mangle)]
pub static KTEST_TEST_WHITELIST: Option<&[&str]> = None;
// SAFETY: The name does not collide with other symbols.
#[unsafe(no_mangle)]
pub static KTEST_CRATE_WHITELIST: Option<&[&str]> = Some(&[
    "os",
]);
```


</details>


```
>>> x/i frame.fde_result.fde.initial_address
   0xffffffff883b03c0 <core::panicking::panic_fmt>:     sub    rsp,0x38
>>> x/i frame.fde_result.fde.initial_address
   0xffffffff883a9810 <core::panicking::panic_const::panic_const_mul_overflow>: push   rax

```


```
set print asm-demangle on

# 设置 log.txt 为日志文件
set logging file log.txt
# 追加模式
set logging overwrite off
# 输出到日志文件
set logging enabled on
# 关闭日志输出
set logging enabled off
```

```
>>> i b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0xffffffff880f1c25 in osdk_test_kernel::__ostd_panic_handler at src/lib.rs:55
        breakpoint already hit 1 time
2       breakpoint     keep y   0xffffffff882e9508 in unwinding::unwinder::_Unwind_RaiseException at src/unwinder/mod.rs:155
        breakpoint already hit 1 time
3       breakpoint     keep y   0xffffffff882e2f45 in unwinding::unwinder::_Unwind_RaiseException::{closure#0} at src/unwinder/mod.rs:157
        breakpoint already hit 1 time
4       breakpoint     keep y   0xffffffff88124507 in unwinding::panicking::catch_unwind::do_catch<unwinding::panic::RustPanic>
                                                   at /root/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/unwinding-0.2.8/src/panicking.rs:61
5       breakpoint     keep y   0xffffffff882e2ff4 in unwinding::unwinder::_Unwind_RaiseException::{closure#0} at src/unwinder/mod.rs:161
        breakpoint already hit 2 times
        silent
        set logging enabled on
        printf "@ frame.fde_result.fde.initial_address = %p\n", frame.fde_result.fde.initial_address
        x/i frame.fde_result.fde.initial_address
        set logging enabled off
6       breakpoint     keep y   0xffffffff882e30b6 in unwinding::unwinder::_Unwind_RaiseException::{closure#0} at src/unwinder/mod.rs:163
        silent
        set logging enabled on
        printf "@ personality = %p\n", personality
        set logging enabled off
7       breakpoint     keep y   0xffffffff882e321a in unwinding::unwinder::_Unwind_RaiseException::{closure#0} at src/unwinder/mod.rs:186
        breakpoint already hit 1 time
        silent
        set logging enabled on
        printf "@ ctx.ra = %p\n", ctx.ra
        x/i ctx.ra
        set logging enabled off
```

```
#x/i personality
>>> x/i (0xffffffff882fb7b0 as *const ())
   0xffffffff882fb7b0 <unwinding::personality::rust_eh_personality>:    sub    rsp,0x98
>>> l unwinding::personality::rust_eh_personality
145         }
146         Ok(EHAction::Terminate)
147     }
148
149     #[lang = "eh_personality"]
150     unsafe fn rust_eh_personality(
151         version: c_int,
152         actions: UnwindAction,
153         _exception_class: u64,
154         exception: *mut UnwindException,
```

## frame & eh_personality

### Before

```
@ frame.fde_result.fde.initial_address = 0xffffffff882e2200
   0xffffffff882e2200 <unwinding::unwinder::with_context<unwinding::abi::UnwindReasonCode, unwinding::unwinder::_Unwind_RaiseException::{closure_env#0}>>:	sub    rsp,0x28
@ frame.fde_result.fde.initial_address = 0xffffffff882e9500
   0xffffffff882e9500 <unwinding::unwinder::_Unwind_RaiseException>:	push   rax
@ frame.fde_result.fde.initial_address = 0xffffffff882f5dc0
   0xffffffff882f5dc0 <unwinding::panicking::begin_panic<unwinding::panic::RustPanic>>:	sub    rsp,0x38
@ frame.fde_result.fde.initial_address = 0xffffffff882e4bc0
   0xffffffff882e4bc0 <unwinding::panic::begin_panic>:	sub    rsp,0x18
@ frame.fde_result.fde.initial_address = 0xffffffff880f0250
   0xffffffff880f0250 <osdk_test_kernel::panic_handler>:	sub    rsp,0x238
@ personality = 0xffffffff882fb7b0
@ frame.fde_result.fde.initial_address = 0xffffffff880f1c20
   0xffffffff880f1c20 <osdk_test_kernel::__ostd_panic_handler>:	push   rax
@ frame.fde_result.fde.initial_address = 0xffffffff8804b020
   0xffffffff8804b020 <os_osdk_bin::panic>:	push   rax
@ frame.fde_result.fde.initial_address = 0xffffffff883b03c0
   0xffffffff883b03c0 <core::panicking::panic_fmt>:	sub    rsp,0x38
@ frame.fde_result.fde.initial_address = 0xffffffff883a9810
   0xffffffff883a9810 <core::panicking::panic_const::panic_const_mul_overflow>:	push   rax
@ frame.fde_result.fde.initial_address = 0xffffffff88243fe0
   0xffffffff88243fe0 <ostd::mm::frame::allocator::FrameAllocOptions::alloc_segment_with<(), ostd::mm::frame::allocator::{impl#1}::alloc_segment::{closure_env#0}>>:	sub    rsp,0x218
@ personality = 0xffffffff882fb7b0
@ frame.fde_result.fde.initial_address = 0xffffffff88135ed0
   0xffffffff88135ed0 <ostd::mm::frame::allocator::FrameAllocOptions::alloc_segment>:	sub    rsp,0x18
@ frame.fde_result.fde.initial_address = 0xffffffff881309c0
   0xffffffff881309c0 <os::tests::max_segment_creation>:	sub    rsp,0x28
@ frame.fde_result.fde.initial_address = 0xffffffff880e8290
   0xffffffff880e8290 <core::ops::function::FnOnce::call_once<fn(), ()>>:	sub    rsp,0x18
@ frame.fde_result.fde.initial_address = 0xffffffff881244c0
   0xffffffff881244c0 <unwinding::panicking::catch_unwind::do_call<fn(), ()>>:	sub    rsp,0x18
@ frame.fde_result.fde.initial_address = 0xffffffff881245a0
   0xffffffff881245a0 <__rust_try>:	sub    rsp,0x18
@ personality = 0xffffffff882fb7b0
```

```
>>> disassemble osdk_test_kernel::__ostd_panic_handler
Dump of assembler code for function osdk_test_kernel::__ostd_panic_handler:
   0xffffffff880f1c20 <+0>:     push   rax

0x000f613d:     DW_TAG_subprogram
                  DW_AT_low_pc  (0xffffffff880f1c20)
                  DW_AT_high_pc (0xffffffff880f1c2a)
                  DW_AT_frame_base      (DW_OP_reg7 EDI)
                  DW_AT_name    ("__ostd_panic_handler")
                  DW_AT_decl_file       ("/asterinas/osdk/deps/test-kernel/src/lib.rs")
                  DW_AT_decl_line       (55)
                  DW_AT_external        (true)
                  DW_AT_noreturn        (true)

00014ad8 0000000000000010 00014adc FDE cie=00000000 pc=ffffffff880f1c20..ffffffff880f1c2a
  DW_CFA_advance_loc: 1 to ffffffff880f1c21
  DW_CFA_def_cfa_offset: 16
```

```
>>> disassemble os_osdk_bin::panic
Dump of assembler code for function os_osdk_bin::panic:
   0xffffffff8804b020 <+0>:     push   rax

0x0000058a: DW_TAG_compile_unit
              DW_AT_producer    ("clang LLVM (rustc version 1.94.0-nightly (1aa9bab4e 2025-12-05))")
              DW_AT_language    (DW_LANG_Rust)
              DW_AT_name        ("src/main.rs/@/eewg0cmcu8pcy3a0wvxwvpo78")
              DW_AT_stmt_list   (0x00000075)
              DW_AT_comp_dir    ("/os/target/osdk/os-test-base")
              DW_AT_low_pc      (0xffffffff8804b020)
              DW_AT_high_pc     (0xffffffff8804b02a)

0x000005da:     DW_TAG_subprogram
                  DW_AT_low_pc  (0xffffffff8804b020)
                  DW_AT_high_pc (0xffffffff8804b02a)
                  DW_AT_frame_base      (DW_OP_reg7 EDI)
                  DW_AT_linkage_name    ("_RNvCs2BPxN8jEfYS_7___rustc17rust_begin_unwind")
                  DW_AT_name    ("panic")
                  DW_AT_decl_file       ("/os/target/osdk/os-test-base/src/main.rs")
                  DW_AT_decl_line       (9)
                  DW_AT_noreturn        (true)

00000030 0000000000000014 00000034 FDE cie=00000000 pc=ffffffff8804b020..ffffffff8804b02a
  DW_CFA_advance_loc: 1 to ffffffff8804b021
  DW_CFA_def_cfa_offset: 16
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
```

```
>>> disassemble core::panicking::panic_fmt
Dump of assembler code for function core::panicking::panic_fmt:
   0xffffffff883b03c0 <+0>:     sub    rsp,0x38

0x00758d53:       DW_TAG_subprogram
                    DW_AT_low_pc        (0xffffffff883b03c0)
                    DW_AT_high_pc       (0xffffffff883b0406)
                    DW_AT_frame_base    (DW_OP_reg7 EDI)
                    DW_AT_linkage_name  ("_RNvNtCsh6ugFidcq7g_4core9panicking9panic_fmt")
                    DW_AT_name  ("panic_fmt")
                    DW_AT_decl_file     ("/root/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/panicking.rs")
                    DW_AT_decl_line     (60)
                    DW_AT_external      (true)
                    DW_AT_noreturn      (true)

000658f8 0000000000000010 000658fc FDE cie=00000000 pc=ffffffff883b03c0..ffffffff883b0406
  DW_CFA_advance_loc: 4 to ffffffff883b03c4
  DW_CFA_def_cfa_offset: 64
```


### after

```
@ frame.fde_result.fde.initial_address = 0xffffffff88262640
   0xffffffff88262640 <unwinding::unwinder::with_context<unwinding::abi::UnwindReasonCode, unwinding::unwinder::_Unwind_RaiseException::{closure_env#0}>>:	sub    rsp,0x28
@ frame.fde_result.fde.initial_address = 0xffffffff88266e40
   0xffffffff88266e40 <unwinding::unwinder::_Unwind_RaiseException>:	push   rax
@ frame.fde_result.fde.initial_address = 0xffffffff8824ee90
   0xffffffff8824ee90 <unwinding::panicking::begin_panic<unwinding::panic::RustPanic>>:	sub    rsp,0x38
@ frame.fde_result.fde.initial_address = 0xffffffff882642b0
   0xffffffff882642b0 <unwinding::panic::begin_panic>:	sub    rsp,0x18
@ frame.fde_result.fde.initial_address = 0xffffffff880ebdf0
   0xffffffff880ebdf0 <osdk_test_kernel::panic_handler>:	sub    rsp,0x228
@ personality = 0xffffffff88255b60
@ frame.fde_result.fde.initial_address = 0xffffffff880ed740
   0xffffffff880ed740 <osdk_test_kernel::__ostd_panic_handler>:	push   rax
@ frame.fde_result.fde.initial_address = 0xffffffff8804b000
   0xffffffff8804b000 <os_osdk_bin::panic>:	push   rax
```

减少了 `core::panicking::panic_fmt` 至 `__rust_try` 的回溯代码。

```
# llvm-dwarfdump-18 target/osdk/os-osdk-bin.qemu_elf
0x0014af62:     DW_TAG_subprogram
                  DW_AT_low_pc  (0xffffffff880ed740)
                  DW_AT_high_pc (0xffffffff880ed74a)
                  DW_AT_frame_base      (DW_OP_reg7 EDI)
                  DW_AT_name    ("__ostd_panic_handler")
                  DW_AT_decl_file       ("/asterinas/osdk/deps/test-kernel/src/lib.rs")
                  DW_AT_decl_line       (55)
                  DW_AT_external        (true)
                  DW_AT_noreturn        (true)
# readelf -wf target/osdk/os-osdk-bin.qemu_elf
00012518 0000000000000010 0001251c FDE cie=00000000 pc=ffffffff880ed740..ffffffff880ed74a
  DW_CFA_advance_loc: 1 to ffffffff880ed741
  DW_CFA_def_cfa_offset: 16
```

```
>>> disassemble os_osdk_bin::panic
Dump of assembler code for function os_osdk_bin::panic:
   0xffffffff8804b000 <+0>:     push   rax

0x0000000b: DW_TAG_compile_unit
              DW_AT_producer    ("clang LLVM (rustc version 1.94.0-nightly (1aa9bab4e 2025-12-05))")
              DW_AT_language    (DW_LANG_Rust)
              DW_AT_name        ("src/main.rs/@/11jl6i1rnza1zj2my20ci1noe")
              DW_AT_stmt_list   (0x00000000)
              DW_AT_comp_dir    ("/os/target/osdk/os-test-base")
              DW_AT_low_pc      (0xffffffff8804b000)
              DW_AT_high_pc     (0xffffffff8804b00a)

0x0000005b:     DW_TAG_subprogram
                  DW_AT_low_pc  (0xffffffff8804b000)
                  DW_AT_high_pc (0xffffffff8804b00a)
                  DW_AT_frame_base      (DW_OP_reg7 EDI)
                  DW_AT_linkage_name    ("_RNvCs2BPxN8jEfYS_7___rustc17rust_begin_unwind")
                  DW_AT_name    ("panic")
                  DW_AT_decl_file       ("/os/target/osdk/os-test-base/src/main.rs")
                  DW_AT_decl_line       (9)
                  DW_AT_noreturn        (true)

00000018 0000000000000014 0000001c FDE cie=00000000 pc=ffffffff8804b000..ffffffff8804b00a
  DW_CFA_advance_loc: 1 to ffffffff8804b001
  DW_CFA_def_cfa_offset: 16
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
```


```
Dump of assembler code for function core::panicking::panic_fmt:
   0xffffffff88302b50 <+0>:     push   rbp

0x0074e8e4:       DW_TAG_subprogram
                    DW_AT_low_pc        (0xffffffff88302b50)
                    DW_AT_high_pc       (0xffffffff88302b7c)
                    DW_AT_frame_base    (DW_OP_reg6 ESI)
                    DW_AT_linkage_name  ("_RNvNtCs3FyYWfWPBng_4core9panicking9panic_fmt")
                    DW_AT_name  ("panic_fmt")
                    DW_AT_decl_file     ("/rustc/1aa9bab4ecbce4859eaad53000f78158ebe2be2c/library/core/src/panicking.rs")
                    DW_AT_decl_line     (60)
                    DW_AT_external      (true)
                    DW_AT_noreturn      (true)

00002be0 000000000000001c 00001060 FDE cie=00001060 pc=ffffffff88302b50..ffffffff88302b7c
  DW_CFA_advance_loc: 1 to ffffffff88302b51
  DW_CFA_def_cfa_offset: 16
  DW_CFA_offset: r6 (esi) at cfa-16
  DW_CFA_advance_loc: 3 to ffffffff88302b54
  DW_CFA_def_cfa_register: r6 (esi)
```

增加 ctx.ra


```
@ frame.fde_result.fde.initial_address = 0xffffffff88262640
   0xffffffff88262640 <unwinding::unwinder::with_context<unwinding::abi::UnwindReasonCode, unwinding::unwinder::_Unwind_RaiseException::{closure_env#0}>>:	sub    rsp,0x28
@ ctx.ra =0xffffffff88266e4d
   0xffffffff88266e4d <unwinding::unwinder::_Unwind_RaiseException+13>:	pop    rcx
@ frame.fde_result.fde.initial_address = 0xffffffff88266e40
   0xffffffff88266e40 <unwinding::unwinder::_Unwind_RaiseException>:	push   rax
@ ctx.ra =0xffffffff8824eef3
   0xffffffff8824eef3 <unwinding::panicking::begin_panic<unwinding::panic::RustPanic>+99>:	add    rsp,0x38
@ frame.fde_result.fde.initial_address = 0xffffffff8824ee90
   0xffffffff8824ee90 <unwinding::panicking::begin_panic<unwinding::panic::RustPanic>>:	sub    rsp,0x38
@ ctx.ra =0xffffffff882642c3
   0xffffffff882642c3 <unwinding::panic::begin_panic+19>:	add    rsp,0x18
@ frame.fde_result.fde.initial_address = 0xffffffff882642b0
   0xffffffff882642b0 <unwinding::panic::begin_panic>:	sub    rsp,0x18
@ ctx.ra =0xffffffff880ec210
   0xffffffff880ec210 <osdk_test_kernel::panic_handler+1056>:	mov    DWORD PTR [rsp+0x1c],eax
@ frame.fde_result.fde.initial_address = 0xffffffff880ebdf0
   0xffffffff880ebdf0 <osdk_test_kernel::panic_handler>:	sub    rsp,0x228
@ personality = 0xffffffff88255b60
@ ctx.ra =0xffffffff880ed74a
   0xffffffff880ed74a:	int3
@ frame.fde_result.fde.initial_address = 0xffffffff880ed740
   0xffffffff880ed740 <osdk_test_kernel::__ostd_panic_handler>:	push   rax
@ ctx.ra =0xffffffff8804b00a
   0xffffffff8804b00a:	int3
@ frame.fde_result.fde.initial_address = 0xffffffff8804b000
   0xffffffff8804b000 <os_osdk_bin::panic>:	push   rax
@ ctx.ra =0xffffffff88302b7c
   0xffffffff88302b7c:	int3
```


## 查看 .eh_frame_hdr

.eh_frame_hdr 是链接器针对 .eh_frame 生成的一个升序索引，而 libunwind 和 gimli 这样的库能够采用二分法快速查找 fde。

像 readelf 这样常见的工具可以解析一部分头部数据，确定 .eh_frame_hdr 存在。

```bash
readelf -l target/osdk/os-osdk-bin.qemu_elf | grep .eh_frame_hdr
   04     .ex_table .ktest_array .init_array .sensitive_io_ports .rodata .eh_frame_hdr .eh_frame .gcc_except_table

llvm-readelf --headers --unwind target/osdk/os-osdk-bin.qemu_elf | less
```

它们输出 .eh_frame 但没有提供 .eh_frame_hdr 的内容，而 eu-readelf 可以同时输出 .eh_frame_hdr 和 .eh_frame 两个段：

```bash
$ sudo apt install -y elfutils
$ eu-readelf --debug-dump=frames target/osdk/os-osdk-bin.qemu_elf | less

Call frame search table section [12] '.eh_frame_hdr':
 version:          1
 eh_frame_ptr_enc: 0x1b (sdata4 pcrel)
 fde_count_enc:    0x3 (udata4)
 table_enc:        0x3b (sdata4 datarel)
 eh_frame_ptr:     0x1b20c (offset: 0x36dac8)
 fde_count:        13888
 Table:
  0xffcb9748 (offset: 0xc000) -> 0x1b228 fde=[    18]
  0xffcb9758 (offset: 0xc010) -> 0x1b240 fde=[    30]
  0xffcb9778 (offset: 0xc030) -> 0x1b258 fde=[    48]
  0xffcb9788 (offset: 0xc040) -> 0x1b26c fde=[    5c]
  0xffcb9a88 (offset: 0xc340) -> 0x1b288 fde=[    78]
  0xffcb9b88 (offset: 0xc440) -> 0x1b2a0 fde=[    90]
  0xffcb9d08 (offset: 0xc5c0) -> 0x1b2c0 fde=[    b0]
  0xffcba048 (offset: 0xc900) -> 0x1b2e4 fde=[    d4]
  0xffcba1e8 (offset: 0xcaa0) -> 0x1b304 fde=[    f4]
  0xffcba518 (offset: 0xcdd0) -> 0x1b328 fde=[   118]
  0xffcba608 (offset: 0xcec0) -> 0x67a6c fde=[ 4c85c]
  0xffcba7f8 (offset: 0xd0b0) -> 0x67a94 fde=[ 4c884]
  0xffcba9f8 (offset: 0xd2b0) -> 0x67ac0 fde=[ 4c8b0]
  0xffcbae68 (offset: 0xd720) -> 0x67ae8 fde=[ 4c8d8]
  0xffcbb218 (offset: 0xdad0) -> 0x67b10 fde=[ 4c900]
  0xffcbb688 (offset: 0xdf40) -> 0x67b38 fde=[ 4c928]
  0xffcbba28 (offset: 0xe2e0) -> 0x67b60 fde=[ 4c950]
  0xffcbbe98 (offset: 0xe750) -> 0x67b88 fde=[ 4c978]
  0xffcbc248 (offset: 0xeb00) -> 0x67bb0 fde=[ 4c9a0]
  ...
  0xfff6de08 (offset: 0x2c06c0) -> 0x6794c fde=[ 4c73c]
  0xfff6de28 (offset: 0x2c06e0) -> 0x67964 fde=[ 4c754]
  0xfff6de48 (offset: 0x2c0700) -> 0x6797c fde=[ 4c76c]
  0xfff6df18 (offset: 0x2c07d0) -> 0x67998 fde=[ 4c788]
  0xfff6df38 (offset: 0x2c07f0) -> 0x679b0 fde=[ 4c7a0]
  0xfff6df58 (offset: 0x2c0810) -> 0x679c8 fde=[ 4c7b8]
  0xfff6dfe8 (offset: 0x2c08a0) -> 0x679e0 fde=[ 4c7d0]
  0xfff6e078 (offset: 0x2c0930) -> 0x679f8 fde=[ 4c7e8]
  0xfff6e0b8 (offset: 0x2c0970) -> 0x67a14 fde=[ 4c804]
  0xfff6e188 (offset: 0x2c0a40) -> 0x67a30 fde=[ 4c820]

Call frame information section [13] '.eh_frame' at offset 0x36dac8:

 [     0] CIE length=20
   CIE_id:                   0
   version:                  1
   augmentation:             "zR"
   code_alignment_factor:    1
   data_alignment_factor:    -8
   return_address_register:  16
   Augmentation data:        0x1b (FDE address encoding: sdata4 pcrel)

   Program:
     def_cfa r7 (edi) at offset 8
     offset r16 (st5) at cfa-8
     nop
     nop

 [    18] FDE length=20 cie=[     0]
   CIE_pointer:              28
   initial_location:         0xffffffff8804b000 <_RNvCs2BPxN8jEfYS_7___rustc17rust_begin_unwind> (offset: 0xc000)
   address_range:            0xa (end offset: 0xc00a)

   Program:
     advance_loc 1 to 0xc001
     def_cfa_offset 16
     nop
```

从 eu-readelf 中同样能阅读到 `readelf -wf` 的内容，但稍微更人性化一些：

```
// after
 [  2be0] FDE length=28 cie=[  1060]
   CIE_pointer:              4192
   initial_location:         0xffffffff88302b50 <_RNvNtCs3FyYWfWPBng_4core9panicking9panic_fmt>
   address_range:            0x2c

   Program:
     advance_loc 1 to 0x1
     def_cfa_offset 16
     offset r6 (esi) at cfa-16
     advance_loc 3 to 0x4
     def_cfa_register r6 (esi)
```

<details>

<summary>AI 解读上面的输出</summary>

这段信息描述的是一个 **FDE (Frame Descriptor Entry)**，它是 DWARF 调试信息（或 `.eh_frame`）中用于**栈回溯（Stack Unwinding）**的核心条目。

下面我为你逐行解析这些数字的含义：

1. 头部定位信息
*   **`[  2be0]`**: 
    这是该 FDE 在 `.eh_frame` 段（或 `.debug_frame` 段）中的**十六进制偏移量**（Offset）。如果你用二进制查看器打开文件，在这个位置就能找到这条数据的开头。
*   **`FDE length=28`**: 
    该 FDE 条目的长度是 28 字节（不包括长度字段本身的 4 字节）。也就是说，这一整块 FDE 数据在磁盘上占用了 32 字节。
*   **`cie=[  1060]`**: 
    该 FDE 关联的 **CIE (Common Information Entry)** 的位置在偏移量 `0x1060` 处。
    *   **为什么要关联 CIE？** 因为很多 FDE 共享相同的基本信息（比如对齐方式、架构寄存器定义等），为了节省空间，把公用信息放在 CIE，FDE 只需要引用它。
*   **`CIE_pointer: 4192`**: 
    这只是上面 `1060` 的十进制表示（$0x1060 = 4192$）。

2. 覆盖的代码范围
*   **`initial_location: 0xffffffff88302b50`**: 
    这个 FDE 负责的代码段的**起始内存地址**。
    *   后面的 `<_RNvNt...core9panicking9panic_fmt>` 是 Rust 编译后的混淆函数名，表示这个 FDE 是为 `core::panicking::panic_fmt` 这个函数准备的。
*   **`address_range: 0x2c`**: 
    这个函数的大小（长度）是 `0x2c` 字节。
    *   **范围计算**：该 FDE 覆盖的有效地址范围是 `0xffffffff88302b50` 到 `0xffffffff88302b7c` ($2b50 + 2c = 2b7c$)。当 CPU 的指令指针 (IP/PC) 处于这个范围内时，回溯器（如 `gimli`）就会使用这个 FDE。

3. 程序指令 (Program: 调用帧指令)
这一部分被称为 **CFI (Call Frame Instructions)**。它像一段微代码，告诉回溯器：**“随着函数代码的执行，栈帧和寄存器是如何变化的？”**

*   **`advance_loc 1 to 0x1`**: 
    地址向前推进 1 个单位（通常是字节）。现在指的是函数起始地址后的第 1 个字节（即 `pc = initial_location + 1`）。
*   **`def_cfa_offset 16`**: 
    **定义 CFA（Canonical Frame Address）**。这里表示：当前栈帧的基准地址（CFA）位于“当前栈指针（SP）+ 16 字节”的位置。这通常发生在函数压栈了某些数据之后。
*   **`offset r6 (esi) at cfa-16`**: 
    告诉回溯器：寄存器 `r6`（在 x86 中是 `esi`）被保存在了地址 `CFA - 16` 的位置。如果你想在回溯时恢复 `esi` 的值，去内存这里找。
*   **`advance_loc 3 to 0x4`**: 
    地址再向前推进 3 个单位。现在来到了函数起始地址后的第 4 个字节（`pc = initial_location + 4`）。
*   **`def_cfa_register r6 (esi)`**: 
    **重要变更**：从现在开始，计算 CFA 的基准不再是栈指针（SP），而是改用寄存器 `r6` (`esi`)。这通常意味着函数已经设置好了**帧指针（Frame Pointer）**。

总结：`gimli` 是如何利用这些数字的？
1.  **查找**：你在 `panic_fmt` 函数里崩了，`gimli` 拿到当前的 PC 指针 `0xffffffff88302b55`。
2.  **匹配**：它去 `.eh_frame_hdr` 里搜（二分法），发现这个地址落在 `2be0` 这个 FDE 的范围内（`2b50` 到 `2b7c` 之间）。
3.  **解析**：`gimli` 读取 `Program` 里的指令。
4.  **计算**：
    *   因为 `2b55` 大于起始地址 + 4，所以 `gimli` 执行完所有的指令。
    *   它知道现在的 `CFA` 在 `esi` 寄存器的位置。
    *   它知道上一层函数的返回地址（Return Address）通常就在 `CFA` 附近的某个偏移量处。
5.  **跳转**：`gimli` 成功找到上一层函数的 PC，实现了一次“栈回溯”。

**简单来说**：这些数字就是一份“**地图**”，告诉调试器在函数运行到某一行时，该去哪里找上一层函数的遗物（寄存器和返回地址）。

</details>


```
>>> p/x address # panic_fmt
$4 = 0xffffffff88302b7b
>>> p/x entry.initial_address
$5 = 0xffffffff882ffa40

eu-readelf 查询 $5 的结果
  0xfff6e188 (offset: 0x2c0a40) -> 0x67a30 fde=[ 4c820] (header 最后一项)
 [ 4c820] FDE length=20 cie=[     0]
   CIE_pointer:              313380
   initial_location:         0xffffffff882ffa40 <_RNvMse_NtNtCs3FyYWfWPBng_4core3num7nonzeroINtB5_7NonZerojE3getCs2rxt17yEbZu_8zerocopy> (offset: 0x2c0a40)
   address_range:            0xa (end offset: 0x2c0a4a)
```


```

Call frame search table section [12] '.eh_frame_hdr':
Call frame information section [13] '.eh_frame' at offset 0x36dac8:
Exception handling table section [14] '.gcc_except_table':
DWARF section [28] '.debug_frame' at offset 0x18dea60:
```
```
线性查表的最后一项 —— 位于 .eh_frame 段
 [ 5a728] FDE length=40 cie=[ 4c838]
   CIE_pointer:              57076
   initial_location:         0xffffffff882fe610 <_RINvCskON4uLgwQod_3log16set_logger_innerNCNvB2_10set_logger0EB2_> (offset: 0x2bf610)
   address_range:            0x142 (end offset: 0x2bf752)
   Augmentation data:        LSDA pointer: 0xffffffff8841a1a4
```

而 panic_fmt 符号位于 .debug_frame 段。

通过 `readelf -S target/osdk/os-osdk-bin.qemu_elf | less` 和对照  panic 为 unwinding 行为的编译产物，发现它们没有
`.debug_frame` 段。

这是因为裸机目标的 target spec （通过以下命令查看）通常是 abort 的 panic 策略，因此支持 unwinding 时，需要重新构建 libcore 
`RUSTCFLAGS="-Cpanic=unwind"` 生成 unwind 代码，把 panic_fmt 放回 .eh_frame 段。

```
rustc -Z unstable-options --print target-spec-json --target x86_64-unknown-none
```
