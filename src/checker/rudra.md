# Rudra

Rudra 仓库： <https://github.com/sslab-gatech/Rudra>

Charon-Rudra 仓库： <https://github.com/AeneasVerif/charon-rudra> （Charon 重写 Rudra 数据源，但未真正实现算法）

## 算法

### Unsafe Dataflow

* unresolvable function: a function whose definitions can't be found without precise type parameters （必须使用精确类型参数才能找到声明的函数）
  * `Vec::<T>::push` 不是 unresolvable function，因为它的实现针对所有 T
  * `<reader as Read>::read()` 是 unresolvable function，因为必须知道 reader 的类型才能找到实现
* The algorithm models six classes of lifetime bypasses:
  * coarse-grained taint tracking to identify panic safety bugs and higher-order invariant bugs 
  * adjustable precision

|   | precision | lifetime bypass      | explanation                            | e.g.               |
|:-:|-----------|----------------------|----------------------------------------|--------------------|
| 1 | high      | uninitialized values | creating uninitialized values          | `Vec::set_len()`   |
| 2 | medium    | duplicate            | duplicating the lifetime of objects    | `ptr::read()`      |
| 3 | medium    | write                | overwriting the memory of a value      | `ptr::write()`     |
| 4 | medium    | copy                 | memcpy()-like buffer copy              | `ptr::copy()`      |
| 5 | low       | transmute            | reinterpreting a type and its lifetime | `mem::transmute()` |
| 6 | low       | ptr-to-ref           | converting a pointer to a reference    |                    |

<details>

<summary> Algorithm 1: Checking unsafe dataflow </summary>

<img src="https://github.com/user-attachments/assets/2ef911c9-e826-4761-925a-089c25d293a4" width="50%" style="display:block;margin:auto;">

</details>


## Rudra 仓库

### 安装

```bash
git clone https://github.com/sslab-gatech/Rudra.git
cd Rudra

# 安装 rudra 和 cargo-rudra 到 ~/.cargo/bin/ 目录
cargo install --path . --locked
```

<details>

<summary>
  <code>--locked</code> 是必须的，因为 `cargo install` 默认重新计算 Cargo.lock。
</summary>

而 libc 需要 1.63 之后的 Rust 编译器，Rudra 使用 `nightly-2021-10-21` 工具链，rustc 版本为 1.58。

```bash
error: failed to compile `rudra v0.1.0 (/.../Rudra)`, intermediate artifacts can be found at `/.../Rudra/target`
Caused by:
  package `libc v0.2.170` cannot be built because it requires rustc 1.63 or newer, while the currently active rustc version is 1.58.0-nightly
```

</details>

### 运行测试

Rudra 有自己的测试脚本，是用 python 写的，因此使用以下命令运行：

```bash
# 设置局部 python 环境，存放于当前目录的 venv 子目录中。
python3 -m venv venv

# 如果你使用 bash，则运行 . venv/bin/activate；更多终端脚本见 venv/bin/ 目录。
. venv/bin/activate.fish

# 在 python 虚拟环境中，所有模块安装仅针对当前目录。
pip install tomlkit

# 设置 Rust 编译器动态库，因为我们直接使用 rudra CLI
export LD_LIBRARY_PATH="/root/.rustup/toolchains/nightly-2021-10-21-x86_64-unknown-linux-gnu/lib:$LD_LIBRARY_PATH"
```

然后执行 test.py，就可以看到测试结果：

```bash
Rudra $ python3 test.py
Make sure you've installed the latest Rudra with `install-[debug|release].sh`
SUCCESS         tests/panic_safety/order_safe_loop.rs
SUCCESS         tests/unsafe_destructor/copy_filter.rs
SUCCESS         tests/panic_safety/order_unsafe_transmute.rs
FALSE-NEGATIVE  tests/panic_safety/pointer_to_ref.rs
SUCCESS         tests/panic_safety/order_safe.rs
SUCCESS         tests/send_sync/wild_channel.rs
SUCCESS         tests/send_sync/no_generic.rs
SUCCESS         tests/panic_safety/order_safe_if.rs
SUCCESS         tests/panic_safety/order_unsafe.rs
SUCCESS         tests/panic_safety/insertion_sort.rs
SUCCESS         tests/panic_safety/order_unsafe_loop.rs
SUCCESS         tests/send_sync/okay_imm.rs
FALSE-POSITIVE  tests/send_sync/okay_channel.rs
SUCCESS         tests/panic_safety/vec_push_all.rs
SUCCESS         tests/unsafe_destructor/ffi.rs
SUCCESS         tests/send_sync/wild_phantom.rs
SUCCESS         tests/send_sync/okay_phantom.rs
SUCCESS         tests/unsafe_destructor/normal2.rs
SUCCESS         tests/send_sync/wild_send.rs
FALSE-POSITIVE  tests/unsafe_destructor/fp1.rs
SUCCESS         tests/unsafe_destructor/normal1.rs
SUCCESS         tests/send_sync/okay_negative.rs
FALSE-POSITIVE  tests/send_sync/sync_over_send_fp.rs
SUCCESS         tests/send_sync/okay_where.rs
SUCCESS         tests/send_sync/okay_ptr_like.rs
SUCCESS         tests/send_sync/okay_transitive.rs
SUCCESS         tests/send_sync/wild_sync.rs
False-negatives: 1/1
False-positives: 3/3
Normal: 23/23
```

这个测试报告虽然统计了 FP （误报）和 FN （漏报），但在 Normal 上没有区分 TP （有问题）和 TN （无问题）。

每个测试文件都有一段 TOML 开头的元信息，用来表示该测试属于 Normal/FP/FN，以及预期的诊断类别。示例：

```toml
# tests/unsafe_destructor/ffi.rs
test_type = "normal"
expected_analyzers = []

# tests/send_sync/wild_send.rs
test_type = "normal"
expected_analyzers = ["SendSyncVariance"]

# tests/send_sync/sync_over_send_fp.rs
test_type = "fp"
expected_analyzers = ["SendSyncVariance"]

# tests/panic_safety/pointer_to_ref.rs
test_type = "fn"
expected_analyzers = []
```

<details>

<summary>【其他细节说明】</summary>


测试脚本 test.py 通过调用 rudra 命令来检测每个 tests 目录下的 .rs 文件，核心步骤：

1. 设置 `LD_LIBRARY_PATH` 环境变量

<details>

<summary><code>LD_LIBRARY_PATH</code> 介绍</summary>

`LD_LIBRARY_PATH` 是一个环境变量，主要用于动态链接器（Dynamic Linker）或加载器（Loader）来查找共享库（Shared Libraries，例如
.so 文件）的路径。它在运行可执行文件时，帮助系统找到所需的动态链接库。

在 Linux 和其他类 Unix 系统中，可执行文件通常依赖于共享库（动态库）。动态链接器在程序运行时加载这些共享库，并将它们链接到程序中。
`LD_LIBRARY_PATH` 是动态链接器用来查找这些共享库的路径之一。

</details>

```bash
export LD_LIBRARY_PATH="/root/.rustup/toolchains/nightly-2021-10-21-x86_64-unknown-linux-gnu/lib:$LD_LIBRARY_PATH"
```

<details>

<summary>如果使用直接 rudra CLI 时没有设置工具链的动态库路径，将看到错误。</summary>

```bash
Rudra $ rudra --crate-type lib tests/panic_safety/insertion_sort.rs
rudra: error while loading shared libraries: librustc_driver-25ea0f19be037cbe.so: cannot open shared object file: No such file or directory
```

```bash
Rudra $ python3 test.py
Make sure you've installed the latest Rudra with `install-[debug|release].sh`
Traceback (most recent call last):
  File "/rust/tmp/repos/Rudra/test.py", line 149, in <module>
    result.get()
  File "/usr/lib/python3.12/multiprocessing/pool.py", line 774, in get
    raise self._value
  File "/usr/lib/python3.12/multiprocessing/pool.py", line 125, in worker
    result = (True, func(*args, **kwds))
                    ^^^^^^^^^^^^^^^^^^^
  File "/rust/tmp/repos/Rudra/test.py", line 83, in run_test
    output = subprocess.run(
             ^^^^^^^^^^^^^^^
  File "/usr/lib/python3.12/subprocess.py", line 571, in run
    raise CalledProcessError(retcode, process.args,
subprocess.CalledProcessError: Command '['rudra', '-Zrudra-enable-unsafe-destructor', '--crate-type', 'lib', 'tests/panic_safety/insertion_sort.rs']' returned non-zero exit status 127.
```

</details>


2. 调用 rudra 命令来检测单独的 rs 文件 —— 这与通常针对 Cargo 项目的 cargo-rudra 不同

```bash
rudra -Zrudra-enable-unsafe-destructor --crate-type lib tests/panic_safety/insertion_sort.rs
```

将看到以下输出：

```text
2025-03-03 15:15:57.135589 |INFO | [rudra-progress] Rudra started
2025-03-03 15:15:57.135949 |INFO | [rudra-progress] UnsafeDestructor analysis started
2025-03-03 15:15:57.136003 |INFO | [rudra-progress] UnsafeDestructor analysis finished
2025-03-03 15:15:57.136012 |INFO | [rudra-progress] SendSyncVariance analysis started
2025-03-03 15:15:57.136105 |INFO | [rudra-progress] SendSyncVariance analysis finished
2025-03-03 15:15:57.136113 |INFO | [rudra-progress] UnsafeDataflow analysis started
2025-03-03 15:15:57.141940 |INFO | [rudra-progress] UnsafeDataflow analysis finished
2025-03-03 15:15:57.142011 |INFO | [rudra-progress] Rudra finished
warning: 2 warnings emitted

Warning (UnsafeDataflow:/ReadFlow/CopyFlow/WriteFlow): Potential unsafe dataflow issue in `insertion_sort_unsafe`
-> tests/panic_safety/insertion_sort.rs:10:1: 22:2
fn insertion_sort_unsafe<T: Ord>(arr: &mut [T]) {
    unsafe {
        for i in 1..arr.len() {
            let item = ptr::read(&arr[i]);
            let mut j = i - 1;
            while j >= 0 && arr[j] > item {
                j = j - 1;
            }
            ptr::copy(&mut arr[j + 1], &mut arr[j + 2], i - j - 1);
            ptr::write(&mut arr[j + 1], item);
        }
    }
}
```

注意，上面的代码片段与终端里的不一样，它对问题区域显示了颜色：

![rudra-report](https://github.com/user-attachments/assets/2b6c04b6-3c5f-4e02-a230-dca6633f2f5b)

但没有说明这里的红色和黄色是什么含义。

3. 其实测试中，上面的诊断是通过 `RUDRA_REPORT_PATH` 环境变量生成到一个 TOML 文件，比如

```bash
$ RUDRA_REPORT_PATH=report.toml rudra -Zrudra-enable-unsafe-destructor --crate-type lib tests/panic_safety/insertion_sort.rs
```

report.toml 文件内容如下：

```toml
[[reports]]
level = 'Warning'
analyzer = 'UnsafeDataflow:/ReadFlow/CopyFlow/WriteFlow'
description = 'Potential unsafe dataflow issue in `insertion_sort_unsafe`'
location = 'tests/panic_safety/insertion_sort.rs:10:1: 22:2'
source = """
fn insertion_sort_unsafe<T: Ord>(arr: &mut [T]) {
    unsafe {
        for i in 1..arr.len() {
            let item = ptr::read(&arr[i]);
            let mut j = i - 1;
            while j >= 0 && arr[j] > item {
                j = j - 1;
            }
            ptr::copy(&mut arr[j + 1], &mut arr[j + 2], i - j - 1);
            ptr::write(&mut arr[j + 1], item);
        }
    }
}
"""
```

注意：source 文本携带 ansi 颜色转义字符，它的结果与截图一致。

</details>

> 如果要编写一个针对检查工具诊断的测试框架，那么就可以参考这里的设计，并进行完善：
> * TP / TN / FP / FN 断言
> * 诊断输出类别断言
> * 诊断 UI 快照测试
> * 编译器动态库设置
> * 检查命令生成（针对 rs 文件而不是 Cargo 项目）




