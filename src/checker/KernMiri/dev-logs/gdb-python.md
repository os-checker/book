# 多进程应用调试

像 `cargo miri` 这样的工具通常是复杂的、嵌套的应用，因为它涉及这样的进程调用：
1. `cargo` 通过参数识别并调用 `cargo-miri`
2. `cargo-miri` 是 `miri` 的调用封装
3. `miri` 是一个动态链接到 librustc 的应用
4. `cargo-miri` 在编译项目期间调用 `cargo build`，从而会调用 `rustc`、`rustup` 等应用

因此，当调试 `cargo miri` 时，需要经过数十个子进程才能达到 miri 进程，并且 cargo、cargo-miri、miri 
这些应用不止一次地运行，这要求 GDB 调试工具具有控制父子进程、父子线程的能力。

GDB 有多种配置和方式来应对多进程调试。这是我使用它们的一些记录。

## `detach-on-fork`

GDB 默认是 `set detach-on-fork on`，这意味着每当 fork 发生，新的子进程将分离 GDB 独立运行。

把它设置为 `off`，则表示 GDB 接手 fork 产生的新进程，从而父子进程都被 GDB 管理起来。

如果父进程阻塞等待子线程，而 GDB 只允许一个进程运行，那么设置了 `detach-on-fork` 会让两个进程相互等待：

```
>>> si
Can not resume the parent process over vfork in the foreground while
holding the child stopped.  Try "set detach-on-fork" or "set schedule-multiple".
clone3 () at ../sysdeps/unix/sysv/linux/x86_64/clone3.S:62
62      in ../sysdeps/unix/sysv/linux/x86_64/clone3.S
```

现代 glibc 的 fork() 底层可能使用 clone3 系统调用，而某些 clone 标志组合会触发类似 vfork 的行为（父进程阻塞直到子进程释放内存空间）。

schedule-multiple on 让 GDB 可以同时恢复父子进程，因此我们需要下面的设置让 GDB 能顺利进入子进程调试。

```
set detach-on-fork off   # 管理子进程
set schedule-multiple on # 调度多个进程
```

示例：使用上面的设置来调试下面的 Rust 代码

```rust
fn main() {
    println!("main thread: start");
    let output = std::process::Command::new("echo")
        .arg("hi")
        .stdout(std::process::Stdio::piped())
        .spawn()
        .unwrap()
        .wait_with_output()
        .unwrap();
    let hi = std::str::from_utf8(&output.stdout).unwrap();
    println!("main thread: {hi}");
}
```

```
>>> run # 自动跳入子进程上运行
process 2910868 is executing new program: /usr/bin/echo
[Inferior 2 (process 2910868) exited normally]
>>> i inferiors # 查看进程情况：echo 进程执行结束，并保持父进程 hi
  Num  Description       Connection           Executable        
  1    process 2910865   1 (native)           /home/gh-zjp-CN/tmp/hi/target/debug/hi 
* 2    <null>                                 /usr/bin/echo     

>>> inferior 1 # 切换到父进程
[Switching to inferior 1 [process 2910865] (/home/gh-zjp-CN/tmp/hi/target/debug/hi)]
[Switching to thread 1.1 (Thread 0x7ffff7f82780 (LWP 2910865))]
#0  0x00007ffff7d107d7 in __GI___wait4 (pid=2910868, stat_loc=0x7fffffffd4e0, options=0, usage=0x0) at ../sysdeps/unix/sysv/linux/wait4.c:30
warning: 30     ../sysdeps/unix/sysv/linux/wait4.c: No such file or directory
>>> i inferiors 
  Num  Description       Connection           Executable        
* 1    process 2910865   1 (native)           /home/gh-zjp-CN/tmp/hi/target/debug/hi 
  2    <null>                                 /usr/bin/echo     
>>> c
Continuing.
main thread: hi

[Inferior 1 (process 2910865) exited normally]
```

# GDB Python API

## gdb 对象

除了官方的 Python API [文档][api]，我们可以查看一个对象的 docstring 介绍，以及它可供调用的方法：

[api]: https://sourceware.org/gdb/current/onlinedocs/gdb.html/Python.html#Python

```
>>> python help(gdb.inferiors)
Help on built-in function inferiors in module _gdb:

inferiors(...)
    inferiors () -> (gdb.Inferior, ...).
    Return a tuple containing all inferiors.
```

```
>>> python print(dir(gdb.inferiors()[0]))
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__getstate__', '__gt__',
 '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__
', '__sizeof__', '__str__', '__subclasshook__', 'architecture', 'arguments', 'clear_env', 'connection', 'connection_num', 'is_valid', 'main_na
me', 'num', 'pid', 'progspace', 'read_memory', 'search_memory', 'set_env', 'thread_from_handle', 'thread_from_thread_handle', 'threads', 'unse
t_env', 'was_attached', 'write_memory']
```

```
>>> python print(gdb.inferiors()[0].num)
1
>>> python print(gdb.inferiors()[0].pid)
2315217
>>> i inferiors 
  Num  Description       Connection           Executable        
* 1    process 2315217   3 (native)           /home/gh-zjp-CN/tmp/hi/target/debug/hi 
```


