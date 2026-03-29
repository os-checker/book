# Miri 分析记录

## 调试与日志

### 使用 gdb 调试

```bash
# 当前进程的命令行参数
>>> info proc cmdline
process 2267234
cmdline = '/home/gh-zjp-CN/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/bin/cargo metadata --no-deps --format-version 1'

>>> info inferiors
  Num  Description       Connection           Executable
  1    <null>                                 /home/gh-zjp-CN/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/bin/cargo
* 2    process 2267234   1 (native)           /home/gh-zjp-CN/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/bin/cargo

```


```
# This GDB supports auto-downloading debuginfo from the following URLs:
# https://debuginfod.ubuntu.com
# Enable debuginfod for this session? (y or [n])
set debuginfod enabled on

# stop at exec syscall (like invoking cargo)
catch exec
commands
  # 获取当前执行的文件路径
  # $_exec_file 是 GDB 的内置变量，保存了最近一次 exec 的文件路径
  silent
  if $_regex($_exec_file, ".*miri$")
    printf ">>> 成功拦截到 Miri: %s\n", $_exec_file
  else
    printf "跳过无关进程: %s\n", $_exec_file
    continue
  end
end

# 发生 fork 时，不要放掉任何一个进程
set detach-on-fork off
set schedule-multiple on

# 默认留在父进程（Cargo）
#set follow-fork-mode parent
# jump to child process
#set follow-fork-mode child

# 忽略 SIGUSR1 信号
handle SIGUSR1 noprint nostop pass
```

/home/gh-zjp-CN/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/bin/cargo-miri runner /home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/os-osdk-bin

>>> i proc cmdline
process 2297411
cmdline = '/home/gh-zjp-CN/.rustup/toolchains/nightly-2025-12-06-x86_64-unknown-linux-gnu/bin/miri --sysroot /home/gh-zjp-CN/.cache/miri --crate-name os_osdk_bin --edition=2024 src/main.rs --diagnostic-width=160 --crate-type bin --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 --check-cfg cfg(docsrs,test) --check-cfg cfg(feature, values()) -C metadata=0d59b05190c8ca8f -C extra-filename=-0f6171dada871360 --out-dir /home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/deps --target x86_64-unknown-linux-gnu -C incremental=/home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/incremental -L dependency=/home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/deps -L dependency=/home/gh-zjp-CN/KMiri/tests/os/target/miri/debug/deps --extern noprelude:alloc=/home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/deps/liballoc-f7053cd07531c1b0.rlib --extern noprelude:compiler_builtins=/home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/deps/libcompiler_builtins-ba64b2db9d19c20b.rlib --extern noprelude:core=/home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/deps/libcore-7e921fa91c0ac277.rlib --extern os=/home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/deps/libos-5cf10299874187da.rlib --extern osdk_frame_allocator=/home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/deps/libosdk_frame_allocator-8397f4f699a53651.rlib --extern osdk_heap_allocator=/home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/deps/libosdk_heap_allocator-32b97c8314b3a4ac.rlib --extern osdk_test_kernel=/home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/deps/libosdk_test_kernel-46ce649a1d7b5614.rlib --extern ostd=/home/gh-zjp-CN/KMiri/tests/os/target/miri/x86_64-unknown-linux-gnu/debug/deps/libostd-be8cf628c4ea0ddf.rlib -Z unstable-options --cfg ktest -C link-arg=-Tmiri.ld -C relocation-model=static -C relro-level=off -C force-unwind-tables=yes --check-cfg cfg(ktest) -C no-redzone=y -C target-feature=+ermsb --'


### 内置的 tracing

[`enter_trace_span`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_const_eval/macro.enter_trace_span.html)

日志阅读 UI：<https://ui.perfetto.dev>

[perfetto](https://perfetto.dev/docs/)

[tracing](https://perfetto.dev/docs/tracing-101) 概念介绍

该功能并不完善，也因为输出太多而很少被使用。
