---
title: AFL++ Fuzzing-Module模糊测试入门
publishDate: 2026-01-03
description: '模糊测试入门，使用AFL++仓库的推荐文档'
tags:
  - fuzzing
  - test
language: '中文'
---

## 1. 环境准备

### 1.1 配置docker

安装参考这一篇：

[Ubuntu 安装 Docker 的方法（基于Ubuntu 24.04 LTS测试） - 星尘的博客 - 博客园](https://www.cnblogs.com/yqbaowo/p/18889717)

镜像站使用的是1ms，貌似能够下载AFL++的image：

```json
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.m.daocloud.io"
  ]
}
```

### 1.2 配置AFL++容器

1. 拉AFL++镜像：`docker pull aflplusplus/aflplusplus`

   然后执行`docker images`应该能看到`aflplusplus/aflplusplus:latest`这个标识

2. 运行AFL++的镜像：`docker run -it --name aflpp aflplusplus/aflplusplus`

   该命令会自动创建一个容器

   - `-it`表示自动进入容器内的终端
   - `--name`表示给这个容器起名叫`aflpp`
   - `aflplusplus/aflplusplus`表示指定了镜像名称

3. 在容器内的终端里进入`/AFLplusplus`文件夹，然后执行`make`命令构建即可

4. 退出容器`exit`

5. 查看容器ID：`docker ps- a`

   该命令会查看当前创建的所有容器（包括正在运行的和停止的）

6. 提交该容器：`docker commit aflpp aflpp-built`

   - `aflpp`：指定容器名
   - `aflpp-built`：指定新镜像名字

   作用是将一个容器提交为一个新镜像

7. 克隆`fuzzing-module`项目：`git clone https://github.com/alex-maleno/Fuzzing-Module.git`

8. 进入`fuzzing-module`项目的根目录，执行命令：

   ```bash
   docker run --rm -it \
     -v "$(pwd)":/target \
     aflpp-built
   ```

   这表示使用`aflpp-built`启动一个容器，将本地的`fuzzing-module`所在目录挂载到容器的`/target`目录中，并在容器退出后自动删除该容器

至此，AFL++容器的配置都已经完成

## 2. 练习1

### 2.1 编译代码

1. 在`fuzzing-module`项目根目录进容器：

   ```bash
   docker run --rm -it \
     -v "$(pwd)":/target \
     aflpp-built
   ```

2. 进入`target`目录：`cd /target`

3. 可以看到原先的`exercisexxx`都在该目录下，进入`exercise1`

4. `mkdir build && cd build`

5. 编译项目，指定AFL++项目构建出的编译器：

   ```bash
   CC=/AFLplusplus/afl-clang-fast CXX=/AFLplusplus/afl-clang-fast++ cmake ..
   make
   ```

### 2.2 运行AFL++

1. 构建种子文件：

   ```bash
   # 回到exercise1目录下，创建seeds目录并进入，将种子文件生成在该目录下
   cd ..
   mkdir seeds && cd seeds
   for i in {0..4}; do dd if=/dev/urandom of=seed_$i bs=64 count=10; done
   cd ..
   cd build
   ```

2. 使用AFL++运行分析目标程序`/target/exercise1/build/simple_crash`：

   ```bash
   /AFLplusplus/afl-fuzz -i /target/exercise1/seeds -o out -m none -d -- /target/exercise1/build/simple_crash
   ```

然后会有一个终端图形界面出来，显示正在运行，当上面的`crashes`数字变更后，就可以强制停止了，生成的文件在`target/exercise1/build/out/default`目录里，其中`/crashes`目录是需要重点关注的文件夹

### 2.3 分析

#### 2.3.1 目录介绍

首先依次解释一下`default`目录下各个文件夹的作用：

1. 结果目录：

   - **`crashes/`**: **（最重要）** 这是一个目录，存放导致目标程序崩溃（例如，段错误 SIGSEGV、中止 SIGABRT 等）的输入文件。你应该首先分析这里的文件，因为它们直接指向了潜在的安全漏洞。

   - **`hangs/`**: 这是一个目录，存放导致目标程序超时（未在指定时间内完成执行）的输入文件。这些输入可能触发了死循环或非常耗时的操作，可能对应于拒绝服务（DoS）漏洞。

2. 语料库和状态：

   - **`queue/`**: 这是一个目录，包含了所有“有趣”的测试用例（也称为语料库）。AFL++ 会将任何能够触发新代码路径（即覆盖新的“边”）的输入文件保存在这里。这些文件是 fuzzer 进行变异的基础。
   - **`fuzz_bitmap`**: 这是一个二进制文件，记录了整个测试过程中所有被触发过的代码路径的“位图”。fuzzer 通过检查新输入是否会在此位图中点亮新的比特位，来判断该输入是否“有趣”。
   - **`fastresume.bin`**: 这是 AFL++ 的一个特性文件。它存储了 `queue` 目录中每个文件的元数据摘要，使得在恢复或重启 fuzzer 时可以非常快速地加载语料库，而无需重新分析每个文件。

3. 统计和监控：

   - **`fuzzer_stats`**: 一个人类可读的文本文件，包含了 fuzzer 运行时的详细统计信息。你可以通过 `cat build/out/default/fuzzer_stats` 查看，内容包括运行时间、执行速度、总路径数、崩溃数、周期数等。这是监控 fuzzer 状态的主要文件。
   - **`plot_data`**: 一个逗号分隔值（CSV）格式的文本文件，用于绘制 fuzzer 运行状态图。它记录了随时间变化的各种统计数据（如路径数、崩溃数等）。你可以使用 `afl-plot` 工具来将此文件可视化，例如：`afl-plot build/out/default /path/to/plot/output`。

4. 配置和信息：

   - **`cmdline`**: 记录了启动本次 fuzzer 时所使用的完整命令行参数。这对于复现测试环境非常有用。
   - **`fuzzer_setup`**: 记录了 fuzzer 启动时的一些关键设置和检测到的环境信息，例如 CPU 亲和性、内存限制等。
   - **`target_hash`**: 包含了目标二进制文件的 SHA256 哈希值。AFL++ 使用它来检测目标程序是否在 fuzzer 运行期间被修改过。

#### 2.3.2 补充模糊测试基本知识

在分析之前，补充一些模糊测试的基本知识：

1. 为什么必须使用 AFL++ 的编译器？

   简而言之：为了**插桩 (Instrumentation)**。

   AFL++ 不是一个“盲目”的 fuzzer。它是一个**覆盖率引导 (Coverage-guided)** 的 fuzzer。这意味着它需要知道一个输入执行了程序的哪些代码路径，从而判断这个输入是否“有趣”（即是否探索了新的代码区域）。如果一个输入触发了新的代码路径，AFL++ 就会将其保留下来，并作为未来变异的基础。

   为了实现这一点，AFL++ 提供了一套编译器包装器（如 `afl-clang-fast++`, `afl-gcc` 等）。当你使用这些编译器来构建你的目标程序时，它们会在编译过程中向你的程序中**注入（或称“插入”）一些额外的代码**。这些被注入的代码非常轻量，主要做一件事：

   - **在程序的每个基本块（或分支跳转）处，插入一小段代码来更新一个共享的内存区域（称为“位图”或 `bitmap`）。**

   这样，当你的程序运行时：

   1. 每执行到一个新的代码分支，插桩代码就会被触发。
   2. 它会根据当前位置和上一个位置计算一个哈希值，并用这个哈希值作为索引，在共享内存位图中对应的位置做一个标记。
   3. `afl-fuzz` 主进程会监控这个位图。如果一个变异后的输入导致位图中出现了之前从未有过的标记，`afl-fuzz` 就知道这个输入发现了一条新路径。

   **结论：** 如果不使用 AFL++ 的编译器进行插桩，`afl-fuzz` 就无法获得代码覆盖率的反馈。它会退化成一个“哑”fuzzer，只能随机生成输入，效率极低，也无法有效地探索程序深处的逻辑。

2. 种子 (Seed) 的作用和内容是什么？

   种子是模糊测试的**起始点**。`afl-fuzz` 不会从零开始完全随机地生成输入，而是会读取你提供的种子文件，将它们作为初始的测试用例集合。然后，它会对这些种子进行各种变异（比特翻转、算术运算、拼接、删除等）来生成新的输入，再去测试目标程序。

   一个好的种子集合可以极大地提升模糊测试的效率，因为：

   - **提供初始覆盖率**：如果种子本身就是合法的、能够被程序正确解析的输入（例如，一个有效的 PNG 图片作为图片解析器的种子），那么 fuzzer 从一开始就能覆盖到程序的核心处理逻辑，而不是浪费时间去“猜”出一个合法的文件格式。
   - **提供变异的“原材料”**：基于一个结构良好的种子进行小幅变异，更有可能产生能通过初始解析并触发更深层逻辑的有效输入。

   种子的内容可以看做是程序的输入

   比如程序接受一个字符串，那就种子的内容就是一个字符串

   程序接受一个64×64的图片，种子的内容就可以是64×64的，数值在0~255之间的二维数组

   **`dd` 命令生成的内容：**

   ```
   for i in {0..4}; do dd if=/dev/urandom of=seed_$i bs=64 count=10; done
   ```

   这个命令的作用是创建 5 个（`seed_0` 到 `seed_4`）种子文件。每个文件包含 `64 * 10 = 640` 字节的**完全随机的数据**，这些数据来源于 `urandom`（Linux 系统中的一个高质量随机数生成器）。

   在这种情况下，生成的种子是**随机的、无结构的二进制数据**。这适用于测试那些期望接收任意二进制流的程序。但对于需要特定文件格式（如 XML, JSON, PNG）的程序，使用完全随机的数据作为种子效果很差。更好的做法是提供一些结构合法、内容简单的真实文件作为种子。

3. `afl-fuzz` 命令和参数的解释

   ```
   afl-fuzz -i [seeds directory] -o out -m none -d -- [path to the executable]
   ```

   这个命令是启动 AFL++ fuzzer 的核心指令。

   - **`afl-fuzz`**: 启动 fuzzer 的主程序。
   - **`-i [full path to your seeds directory]`**: 指定**输入（input）**目录，也就是你存放种子文件的目录。`afl-fuzz` 会从这里加载初始测试用例。
   - **`-o out`**: 指定**输出（output）**目录。所有 fuzzer 的发现和状态都会保存在这个名为 `out` 的目录中，包括我们之前讨论的 `crashes/`, `queue/`, `fuzzer_stats` 等。
   - **`-m none`**: 设置内存限制（**memory limit**）。`none` 表示不为目标程序设置内存限制。这通常在你知道程序本身内存占用较大，或者在受控环境（如 Docker）中已经有全局内存限制时使用，以避免 fuzzer 因误判而频繁杀死正常运行的子进程。默认情况下，AFL++ 会设置一个比较严格的内存限制。
   - **`-d`**: 启用**“确定性”变异阶段（deterministic** stages）。在进行完全随机的“havoc”变异之前，AFL++ 会先尝试一些简单的、可预测的变异策略，如按位翻转、整数加减等。这是一个非常高效的阶段，通常能很快找到浅层的 bug。`-d` 标志可以跳过这个阶段，直接进入随机性更强的 havoc 阶段。在某些情况下，如果种子文件非常大或者确定性变异阶段耗时过长，可以使用 `-d` 来加速。但在大多数情况下，建议保留确定性阶段。
   - **`--`**: 这是一个分隔符。它告诉 `afl-fuzz`，后面的所有参数都属于要执行的**目标程序命令**，而不是 `afl-fuzz` 自身的参数。
   - **`[full path to the executable]`**: 你要测试的目标程序的路径和名称。`afl-fuzz` 会反复执行这个程序。AFL++ 通常会通过 `@@` 来指定输入文件的位置，例如 `/path/to/my_app @@`。如果省略 `@@`，AFL++ 默认会通过标准输入（stdin）将测试用例喂给目标程序。

#### 2.3.3 调试程序找出bug

1. 文件解释：

   导致程序崩溃的文件都在`build/out/default/crashes`下，其文件名即各种元信息的组合：

   以 `id:000000,sig:06,src:000004,time:160,execs:116,op:flip4,pos:7` 为例：

   - `id:000000`: 崩溃的唯一标识符，从 0 开始递增。
   - `sig:06`: **（关键信息）** 导致程序终止的信号编号。`06` 代表 `SIGABRT` (Abort)，通常由断言失败 `assert()` 或其他库检测到的内部错误触发。另一个常见的是 `sig:11`，代表 `SIGSEGV` (Segmentation Fault)，即段错误，通常由无效的内存访问（如空指针解引用、越界读写）引起。
   - `src:000004`: 源输入文件的 ID。这个崩溃是由 `queue/id:000004` 文件经过变异得到的。
   - `time:160`: 发现此崩溃时，fuzzer 已经运行的时间（毫秒）。
   - `execs:116`: 发现此崩溃时，fuzzer 已经执行的总次数。
   - `op:flip4`: 产生这个崩溃的最后一个变异操作是 `flip4`（翻转了文件中的 4 个比特位）。
   - `pos:7`: 变异操作发生在源文件的偏移量 7 处。

   文件内容即导致目标崩溃的内容

2. gdb调试程序

   进入`/build`目录下

   - 开启调试：`gdb --args ./simple_crash`
   - 在 GDB 提示符 `(gdb)` 后，输入 `run` 命令并重定向输入：`run < build/out/default/crashes/id:000000,sig:06,src:000004,time:160,execs:116,op:flip4,pos:7`
   - **分析崩溃点**：
     程序会在崩溃处停下来。GDB 会显示导致崩溃的代码行。然后你可以使用以下命令来检查程序状态：
     - `bt` (backtrace): 显示函数调用栈。这是最有用的命令，可以让你看到函数是如何一步步调用到崩溃点的。
     - `p <variable>` (print): 打印变量的值。
     - `info registers`: 显示寄存器的值。
     - `info locals`: 显示当前栈帧的局部变量。
     - `x/i $pc`: 查看当前指令。

通过源码和接收的字符串来看，程序接受一行字符串，如果该字符串有效范围内的的第一个字符和最后一个字符都是`\x00`，或者字符串内有递增的数字字符出现（比如"0@1"），就会崩溃

## 3. 练习2

和练习1没什么区别，就是封装了一个类，在`interact`函数里面接收标准输入流

- 初始有两个乘员，`Captain`，`CoPilot`
- `t`：飞机有乘员则起飞
- `l`：飞机有乘员则降落
- `h`：增加一个乘员
- `f`：减少一个乘员

指定`l`命令时，如果没有乘员就`abort`

## 4. 练习3

该练习主要介绍`slice-fuzzing`的办法，人话就是只针对部分代码片段进行模糊测试

`exercise3`中给了一个例子：

```c++
/*
 *
 *
 * This file isolates the Specs class and tests out the 
 * choose_color function specifically.
 * 
 * 
 * 
 */

#include "specs.h"

int main(int argc, char** argv) {
    // In order to call any functions in the Specs class, a Specs
    // object is necessary. This is using one of the constructors
    // found in the Specs class.
    Specs spec(505, 110, 50);
    // By looking at all the code in our project, this is all the 
    // necessary setup required. Most projects will have much more
    // that is needed to be done in order to properly setup objects.

    // This section should be in your code that you write after all the 
    // necessary setup is done. It allows AFL++ to start from here in 
    // your main() to save time and just throw new input at the target.
    #ifdef __AFL_HAVE_MANUAL_CONTROL
        __AFL_INIT();
    #endif

    spec.choose_color();
    //spec.min_alt();

    return 0;
}
```

该代码独立`main.cpp`，在`cmake`中被额外添加构建可执行文件的规则，这相当于主动编写模糊测试驱动程序，在这个程序中，只去测试`Specs`类的`choose_color()`功能, 然后**连接 Fuzzer**：它包含接收 fuzzer 输入并传递给目标函数的逻辑（这部分通常是隐式的，例如目标函数从标准输入读取数据）

**AFL++持久模式：**

标准的模糊测试流程中：AFL++ 每测试一个输入，都需要：

1. 启动一个全新的进程。
2. 在那个新进程里，从`main`函数开始完整地执行一遍程序。
3. 程序执行完毕后，销毁进程。

如果在 fuzz 的关键步骤前面有诸如“载入配置文件”等步骤，仍然会造成效率浪费。因此，我们可以自行选择 fuzz 入口，然后添加以下代码：

```c++
#ifdef __AFL_HAVE_MANUAL_CONTROL
  __AFL_INIT();
#endif
```

该代码之前的部分只被执行一次，之后的部分才是测试的重点，会不断执行，以此来提高fuzzing的效率

**自定义切片代码：**

类似给出的样例，可以创建cpp文件来测试需要的函数：

```c++
#include "specs.h"

int main(int argc, char** argv) {
    Specs spec(505, 110, 50);
    
    #ifdef __AFL_HAVE_MANUAL_CONTROL
        __AFL_INIT();
    #endif

    spec.fuel_cap();

    return 0;
}
```

在`cmake`添加构建规则，然后指定fuzz的目标即可
