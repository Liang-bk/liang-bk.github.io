---
title: 'linux环境下cmake + clangd + lldb搭建cpp开发环境'
publishDate: '2025-11-03'
updatedDate: '2025-11-04'
description: '快速搭建现代cpp环境'
tags:
  - modern c++
  - tutorial
language: '中文' 
---

# linux环境下cmake + clangd + lldb搭建cpp开发环境

## 基础环境

- wsl2
- ubuntu22
- ubuntu24

以上均可（其他没试过，不过理论上都可以）

## 源配置

如果要配置多编译器环境或者更想用**gcc/g++**，这里的源配置主要针对需要下载高等级的gcc版本

由于一些低版本的ubuntu（如22）默认下载gcc11版本，如果想要下载更高的版本，12以及往上，可以使用下面两条命令：

```bash
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo add-apt-repository ppa:ubuntu-toolchain-r/ppa
```

然后可以安装高版本的gcc和g++

```bash
apt search gcc-12
# apt search gcc-13
sudo apt install gcc-12
sudo apt install g++-12
```

不过由于是国外源，下载的会比较慢，需要额外配置镜像（这个镜像不是`/etc/apt/sources.list`文件）

执行上面两条`add-apt-repository`命令后，查看`/etc/apt/sources.list.d`目录：

```bash
ll /etc/apt/sources.list.d
```

如下图所示（一个ppa，一个test）：

![](./source1.png)

以`ppa-jammy.list`为例，里面的内容原本是：

```bash
deb https://ppa.launchpadcontent.net/ubuntu-toolchain-r/ppa/ubuntu/ jammy main 
# deb-src https://ppa.launchpadcontent.net/ubuntu-toolchain-r/ppa/ubuntu/ jammy main
```

使用如下命令换为中科大的源（最后两个参数是`ppa`和`test`两个文件名）：

```bash
sudo sed -i 's|https://ppa.launchpadcontent.net|https://launchpad.proxy.ustclug.org|g' /etc/apt/sources.list.d/ubuntu-toolchain-r-ubuntu-ppa-jammy.list /etc/apt/sources.list.d/ubuntu-toolchain-r-ubuntu-test-jammy.list
```

然后执行：

```bash
sudo apt update
sudo apt install gcc-12 g++-12
```

就会快很多了

## gcc/g++版本切换（可选）

如果电脑下有多个gcc/g++，想要切换版本，首先查看各个gcc/g++的版本号：

```bash
ls -l /usr/bin | grep gcc
```

如下图：

![](./gcc-version.png)

使用`update-alternatives`命令设置各个gcc/g++的优先级：

```bash
# 13的权值是100, 11是80...
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 100 --slave /usr/bin/g++ g++ /usr/bin/g++-13 --slave /usr/bin/gcov gcov /usr/bin/gcov-13
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 80 --slave /usr/bin/g++ g++ /usr/bin/g++-11 --slave /usr/bin/gcov gcov /usr/bin/gcov-11
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 60 --slave /usr/bin/g++ g++ /usr/bin/g++-10 --slave /usr/bin/gcov gcov /usr/bin/gcov-10
```

设置完毕后再执行`gcc -v`就能看到当前是`gcc-13`

如果想要切换版本：

```bash
sudo update-alternatives --config gcc
```

如下图，选择对应版本即可：

![](./gcc-choose.png)

## llvm配置

国外的网址：[apt llvm](https://apt.llvm.org/)

找到下图中的脚本命令：

![](./apt-llvm-script.png)

和版本：

![](./apt-llvm-version.png)

（我安装的时候stable版本是20）

但不要执行，同样是由于国外源太慢的问题，这次换清华源：[tsinghua apt llvm](https://mirrors4.tuna.tsinghua.edu.cn/help/llvm-apt/)

![](./apt-llvm-script-20.png)

在`LLVM版本`选为20（一般选stable版本就对了）

将上面的命令逐行粘贴到终端执行：

```bash
sudo wget https://mirrors.tuna.tsinghua.edu.cn/llvm-apt/llvm.sh
sudo chmod +x llvm.sh
sudo ./llvm.sh all -m https://mirrors.tuna.tsinghua.edu.cn/llvm-apt
```

之后再安装`libstdc++`，配置`gcc/g++`使用

```bash
sudo apt-get install libc++-20-dev libc++abi-20-dev
```

这样就安装好了llvm全套（`clang/clang++/clangd/lldb`）和`libc++`

## cmake配置

不推荐直接从apt下载cmake，版本一般比较旧

一般从源码安装或者作者仓库中拉取，具体参考下面这篇文章：

[ubuntu 22.04环境中cmake安装 - 知乎](https://zhuanlan.zhihu.com/p/694017813)

## vscode配置

插件安装列表（自行安装），还有一个Test Explorer UI插件没有截进来：

![](./vscode-plug.png)



### clangd

#### clangd path

刚下载完clangd，会弹窗提示没有下载clangd-server，这时候不要点击，直接去clangd设置中：

![](./clangd-enter-setting.png)

不是用户的设置而是wsl2或者虚拟机的设置中，找到`Clangd: Path`选项，之前下载`llvm`组件时已经下载了`clangd-<version>`（`<version>`是stable的版本号），直接填入到选项中：

![](./clangd-path.png)

#### .clangd配置系统文件寻找路径

如果不介意头文件内部爆红的话，直接在项目目录下创建`.clangd`文件，然后填入下面的内容：

```yaml
CompileFlags:
  Add:
    - "-stdlib=libc++"  # 告诉编译器使用 libc++
    - "-I/usr/lib/llvm-20/include/c++/v1" # clang++的目录
    - "-ferror-limit=0"
  Compiler: clang++-20
```

如果不使用clang++，只用gcc，可以这么写（不需要配置`-stdlib`，因为linux下默认用`libstdc++`）：

```yaml
CompileFlags:
  Add:
    - "-I/usr/include/c++/13"
    - "-ferror-limit=0"
  Compiler: g++-13
```

什么是内部爆红？如下图：

![](./clangd-failed-working.png)

项目的文件在`#include<string>`时不会爆红，如果进入系统`string`头文件后就会莫名其妙出现一堆`file not found with <angled> include;`的错误，**不影响库使用，也不影响编译和运行**

#### config.yaml配置（针对clang++）

如果有强迫症，可以在`~/.config/clangd/`目录下（没有中间目录就创建）创建`config.yaml`文件，并填入：

```yaml
CompileFlags:
  Add:
    - "I/usr/lib/llvm-20/include/c++/v1"
    - "-stdlib=libc++"
    - "-ferror-limit=0"
```

同时不在项目中配置`.clangd`文件（缺点是clangd对所有的项目都会使用clang++的目录分析了，要改必须去`~/.config/clangd/config.yaml`中改）

至此，clangd的配置就完成了一半

### codelldb

#### 安装vsix

下载插件后，可能会报错，弹窗`open ... url`，点击后在浏览器下载一个`.vsix`文件

然后在扩展中安装：

![](./codelldb-install.png)

#### 调试

配置好cmake tools之前的cmake项

打断点，按`F5`调试，一开始会报错，让创建`launch.json`文件，跟着步骤创建：

```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug",
            "program": "${workspaceFolder}/<executable file>",
            "args": [],
            "cwd": "${workspaceFolder}"
        }
    ]
}
```

把`"program"`项换为可执行文件的位置，就可以正常调试了

### cmake

#### cmake path

在CMake插件中的设置中，找到`Cmake: Cmake Path`选项，设置为cmake的可执行文件路径：

![](./cmake-path.png)

#### cmake快速创建项目

- 创建一个文件夹，然后使用vscode打开
- `Ctrl + Shift + P`唤出命令面板
- 输入`cmake:quick`，找到`CMake: Quick Start`
- 按提示步骤完成剩余操作

#### CMAKE_EXPORT_COMPILE_COMMANDS

使用`git`下载一个cmake项目，比如`fmt: git clone https://github.com/fmtlib/fmt.git`

如果要使用clang++，则找到项目根目录下的`CMakeLists.txt`，加入：

```cmake
# 生成 compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

上述命令会在`build`目录下生成`compile_commands.json`，clangd会根据这个文件生成索引，存放在项目的`.cache`目录下，如果没有，可能不会正常工作

至此，clangd应该能够正常工作了

#### cmake tools

使用过clion和vs2022都清楚有比较方便的切换启动哪个可执行程序的功能，借助cmake tools插件，可以做到这一点，修改`launch.json`（主要是`"program"`项）：

```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug",
            "program": "${command:cmake.launchTargetPath}",
            "args": [],
            "cwd": "${workspaceFolder}"
        }
    ]
}
```

在vscode左侧cmake-tools的图标中，选择启动和调试的目标即可：

![](./cmake-tools-auto.png)

### unit-test

安装`C++ TestMate`和`Test Explorer UI`后，在左侧测试按钮中可以进行对应的测试：

![](./unit-test.png)

在编译后也可以在对应的测试文件中直接使用图形化运行按钮进行测试

但是测试文件（指可执行文件）的识别默认为build或者out目录下的带有`test`名的文件，所以在生成可执行文件时要注意正确的命名：

![](./test-name.png)

## windows下环境配置

请参考[VSCode C/C++开发环境配置 Clang/LLVM+LLDB+CMake](https://www.ispellbook.com/post/VSCodeCppDevConfig)

## 常见问题

### codelldb调试无法正常显示STL内容

#### 问题描述

比如源码为：

```c++
struct node {
    unordered_map<int, int> map;
};
int main() {
    node node;
    node.map.insert({1, 1});
    return 0
}
```

显示的是：

![](./code-lldb-bug.png)

可以看到内嵌到`struct`或者`class`内的STL组件可能无法正常显示

#### 解决方案

https://github.com/vadimcn/codelldb/issues/707

插件作者说lldb不能支持MSVC STL，猜测是由于linux下使用g++ STL导致的问题（cmake会默认链接`libstdc++`）

修改`CMakeLists.txt`，使用llvm组件（这些组件在llvm配置时应该就配置完成了）：

```cmake
# 设置编译器
set(CMAKE_CXX_COMPILER clang++-20)
# 添加编译标志
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
# 设置链接标志
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -stdlib=libc++ -lc++abi")
```

重新编译，再调试：

![](./code-lldb-bug-fixed.png)

ps：在linux上使用llvm工具链还是不太方便，修改了好多配置，不想折腾还是默认使用g++和gdb比较好

### clangd设置代码补全

#### 问题描述

clangd默认配置在代码补全的时候会把模板或函数的所有参数都补全出来，例如，当你输入了

```c++
std::set
```

并试图使用clangd提供的代码补全时，它会补全为：

```c++
std::set<class, class, class>
```

可以使用`Tab`一个一个填充或删除参数，但这样比较麻烦

#### 解决方案

在`.vscode`文件夹下新增`settings.json`，内容为：

```json
{
    "clangd.arguments": [
        "--function-arg-placeholders=false",
        "--completion-style=bundled"
    ]
}
```

这样会影响在当前项目目录下`clangd`的补全功能，当输入`std::set`时，会补全为`std::set<>`

也可以直接在`clangd`插件的设置中找到`arguments`项，设置上面两个参数，这样会影响所有目录下的补全行为

## 参考

- [如何在 Ubuntu 中安装和切换多版本 GCC 编译器 - 系统极客](https://www.sysgeek.cn/ubuntu-install-gcc-compiler/)

- [ubuntu 22.04 切 gcc/g++ 版本 - 知乎](https://zhuanlan.zhihu.com/p/639332690)

- [WSL下 C++ 使用 clang19 编译器并配置 vscode 的 clangd 服务器_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1gPNJe6E3z/?spm_id_from=333.1387.top_right_bar_window_history.content.click&vd_source=b93859dd8360e859dab85c63d1b91f9f)

- [基于VSCode+Clangd+lldb搭建Linux C++环境_vscode lldb-CSDN博客](https://blog.csdn.net/weixin_45896211/article/details/135310728)

- [ubuntu 22.04环境中cmake安装 - 知乎](https://zhuanlan.zhihu.com/p/694017813)

- [VSCode C/C++开发环境配置 Clang/LLVM+LLDB+CMake](https://www.ispellbook.com/post/VSCodeCppDevConfig)

