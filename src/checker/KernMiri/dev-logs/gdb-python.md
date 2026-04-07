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
# 管理子进程
set detach-on-fork off
# 调度多个进程
set schedule-multiple on
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

## `catch exec`

上面的例子成功在子进程发起之后切换并执行。但这并不足够，因为我们需要自动切换但不自动执行。

可以利用 `catch exec` 设置一个“断点”，它捕捉 exec 系统调用，并在切换子进程之后暂停，然后我们能够正常对它调试，比如查询符号、设置断点、运行 continue 等

```
>>> set detach-on-fork off
>>> set schedule-multiple on
>>> catch exec
Catchpoint 1 (exec)
```
```
>>> r
Starting program: /home/gh-zjp-CN/tmp/hi/target/debug/hi 

main thread: start
[New inferior 2 (process 2914989)]
[Switching to process 2914989]

Thread 2.1 "echo" hit Catchpoint 1 (exec'd /usr/bin/echo), 0x00007ffff7fe4540 in _start () from /lib64/ld-linux-x86-64.so.2
>>> i inferiors # 注意 echo 进程有连接状态，而不是 null
  Num  Description       Connection           Executable        
  1    process 2914985   1 (native)           /home/gh-zjp-CN/tmp/hi/target/debug/hi 
* 2    process 2914989   1 (native)           /usr/bin/echo     
>>> f # 查看当前栈帧，我们停在子进程的程序入口
#0  0x00007ffff7fe4540 in _start () from /lib64/ld-linux-x86-64.so.2
```

## `follow-fork-mode`

在所有设置都是默认情况下， `follow-fork-mode` 为 parent，这意味着程序发起子进程之后，GDB 不会进入子进程执行。

我们可以设置它为 child，每当 fork 发生，让 GDB 跟随子进程执行：

```gdb
>>> set follow-fork-mode child

>>> r
Starting program: /home/gh-zjp-CN/tmp/hi/target/debug/hi 
main thread: start
[Attaching after Thread 0x7ffff7f82780 (LWP 2968237) vfork to child process 2968252]
[New inferior 2 (process 2968252)]
[Detaching vfork parent process 2968237 after child exec]
[Inferior 1 (process 2968237) detached]
process 2968252 is executing new program: /usr/bin/echo
warning: could not find '.gnu_debugaltlink' file for /usr/bin/echo
main thread: hi

[Inferior 2 (process 2968252) exited normally]

>>> i inferiors 
  Num  Description       Connection           Executable        
  1    <null>                                 /usr/bin/echo     
* 2    <null>                                 /usr/bin/echo     
```

可以看到只设置 `follow-fork-mode` 为 `child`，GDB 在子进程发生之后直接进入并执行，并解除对父进程的控制，最终父子进程都运行结束，中途没有停下。

设置 `catch exec` 可在进入子进程之后停下，但父进程依然游离于 GDB：

```
>>> set follow-fork-mode child

>>> catch exec

>>> r
Starting program: /home/gh-zjp-CN/tmp/hi/target/debug/hi 
main thread: start
[Attaching after Thread 0x7ffff7f82780 (LWP 2997061) vfork to child process 2997064]
[New inferior 2 (process 2997064)]
[Detaching vfork parent process 2997061 after child exec]
[Inferior 1 (process 2997061) detached]
process 2997064 is executing new program: /usr/bin/echo
[Switching to process 2997064]

Thread 2.1 "echo" hit Catchpoint 1 (exec'd /usr/bin/echo), 0x00007ffff7fe4540 in _start () from /lib64/ld-linux-x86-64.so.2
>>> i inferiors 
  Num  Description       Connection           Executable        
  1    <null>                                 /usr/bin/echo     
* 2    process 2997064   1 (native)           /usr/bin/echo     
```

但这种模式只适用于线性的进程调试，对我们不太有用，因为
* 遇到子进程就执行
* 丢失对父进程的控制权

而我们需要的是：
* 保持 GDB 对 `cargo miri` 的父进程控制
* 遇到子进程，检查是否为 miri：
  * 是：停下
  * 不是：执行到结束，返回父进程继续执行


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

# 安装最新的 GDB

像 events.selected_context 接口需要 GDB 版本为 15.1 以上，而 apt 安装的 GDB 比较旧，因此需要源码安装。

查看 [最新的](https://ftp.gnu.org/gnu/gdb/) 版本，本地非 root 用户安装 GDB 命令：

```bash
wget https://ftp.gnu.org/gnu/gdb/gdb-17.1.tar.xz
tar -xf gdb-17.1.tar.xz
rm ./gdb-17.1/build/ -rf
rm ~/.local/bin/gdb/ -rf
mkdir ./gdb-17.1/build
cd ./gdb-17.1/build

export MY_GDB_ROOT=$HOME/.local/bin/gdb

wget https://ftp.gnu.org/gnu/ncurses/ncurses-6.6.tar.gz
tar -xf ncurses-6.6.tar.gz
cd ncurses-6.6
./configure --prefix=$MY_GDB_ROOT \
  --with-shared \
  --enable-widec \
  --with-termlib \
  --enable-pc-files \
  --with-pkg-config-libdir=$MY_GDB_ROOT/lib/pkgconfig
make -j$(nproc)
make install
cd ..

wget https://ftp.gnu.org/gnu/readline/readline-8.2.tar.gz
tar -xf readline-8.2.tar.gz
cd readline-8.2
./configure --prefix=$MY_GDB_ROOT \
  --with-curses \
  LDFLAGS="-L$MY_GDB_ROOT/lib" \
  CPPFLAGS="-I$MY_GDB_ROOT/include" \
  SHLIB_LIBS="-lncursesw"
make SHLIB_LIBS="-L$MY_GDB_ROOT/lib -lncursesw" -j$(nproc)
make install
cd ..

wget https://ftp.gnu.org/gnu/gmp/gmp-6.3.0.tar.xz
tar -xf gmp-6.3.0.tar.xz
cd gmp-6.3.0
./configure --prefix=$MY_GDB_ROOT
make -j$(nproc)
make install
cd ..

wget https://ftp.gnu.org/gnu/mpfr/mpfr-4.2.2.tar.xz
tar -xf mpfr-4.2.2.tar.xz
cd mpfr-4.2.2
./configure --prefix=$MY_GDB_ROOT --with-gmp=$MY_GDB_ROOT
make -j$(nproc)
make install
cd ..

wget https://github.com/libexpat/libexpat/releases/download/R_2_7_5/expat-2.7.5.tar.xz
tar -xf expat-2.7.5.tar.xz
cd expat-2.7.5
./configure --prefix=$MY_GDB_ROOT
make -j$(nproc)
make install
cd ..

git clone https://github.com/intel/libipt.git
cd libipt
mkdir build && cd build
cmake .. \
  -DCMAKE_INSTALL_PREFIX=$MY_GDB_ROOT \
  -DCMAKE_INSTALL_LIBDIR=lib \
  -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
make install
cd ../..

# liblzma
wget https://github.com/tukaani-project/xz/releases/download/v5.8.2/xz-5.8.2.tar.xz
tar -xf xz-5.8.2.tar.xz
cd xz-5.8.2
./configure --prefix=$MY_GDB_ROOT --disable-doc
make -j$(nproc)
make install
cd ..

export LDFLAGS="-L$MY_GDB_ROOT/lib -L$MY_GDB_ROOT/lib64 -Wl,-rpath,$MY_GDB_ROOT/lib -Wl,-rpath,$MY_GDB_ROOT/lib64"
export CPPFLAGS="-I$MY_GDB_ROOT/include"
export CFLAGS="-O2 -I$MY_GDB_ROOT/include"
export CXXFLAGS="-O2 -I$MY_GDB_ROOT/include"

../configure --prefix=$MY_GDB_ROOT \
  --with-python=$(which python3) \
  --enable-targets=all \
  --enable-64-bit-bfd \
  --with-python=python3 \
  --with-guile=no \
  --enable-tui \
  --with-source-highlight \
  --with-expat \
  --with-system-readline \
  --with-libipt-prefix=$MY_GDB_ROOT \
  --with-libreadline-prefix=$MY_GDB_ROOT \
  --with-libgmp-prefix=$MY_GDB_ROOT \
  --with-libmpfr-prefix=$MY_GDB_ROOT \
  --with-libexpat-prefix=$MY_GDB_ROOT \
  --with-liblzma-prefix=$MY_GDB_ROOT \
  --enable-sim \
  LDFLAGS="$LDFLAGS" \
  CPPFLAGS="$CPPFLAGS" \
  CFLAGS="$CFLAGS" \
  CXXFLAGS="$CXXFLAGS"
# --with-intel-pt \

make -j$(nproc)
make install
```

添加本用户 GDB 动态库和路径到环境变量：

```bash
# ~/.bashrc
LOCAL_BIN=/${HOME}/.local/bin
GDB_HOME="$LOCAL_BIN/gdb"
export PATH=${GDB_HOME}/bin:$PATH
export LD_LIBRARY_PATH=${GDB_HOME}/lib:$LD_LIBRARY_PATH
```

# 其他资料

* [](http://events17.linuxfoundation.org/sites/events/files/slides/Debugging%20the%20Linux%20Kernel%20with%20GDB.pdf)
* [Guile](https://www.gnu.org/software/guile/)
  * [A Scheme/Guile Primer](https://files.spritely.institute/papers/scheme-primer.html)
  * [Guix: A most advanced operating system](https://web.archive.org/web/20210528220842/https://ambrevar.xyz/guix-advance/index.html)
  * [Guile 问答 (gemini)](https://aistudio.google.com/app/prompts?state=%7B%22ids%22:%5B%221WmgQ2ek-GYHdICBtLmr5AeI_R8_ipmBo%22%5D,%22action%22:%22open%22,%22userId%22:%22107770117539303321065%22,%22resourceKeys%22:%7B%7D%7D&usp=sharing)
