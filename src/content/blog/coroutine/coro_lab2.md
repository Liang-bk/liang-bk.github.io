---
title: c++20制作简易协程库2
publishDate: 2025-11-14 22:00:00
description: ''
tags:
  - modern cpp
  - coroutine
  - tutorial
language: '中文'
---

## io_uring简介

### 安装

github克隆liburing项目：

`git clone https://github.com/axboe/liburing.git`

编译并安装：

```bash
cd liburing
./configure --cc=gcc --cxx=g++
make -j
make liburing.pc
sudo make install
```

在`usr/include/`目录下找到`liburing`即为成功

### 基本使用

liburing提供了io_uring简化后的api调用，类似epoll的工作流程：

- 初始化io_uring：`io_uring_queue_init()`
- 从请求队列中获取一个SQE：`io_uring_get_sqe()`
- 填充读写请求：`io_uring_prep_read() / write()`
- 提交请求：`io_uring_submit()`
- 等待结果：`io_uring_wait_cqe()`
- 标记结果已处理：`io_uring_cqe_seen()`

#### 核心数据结构

1. io_uring：

   类似`epoll_create()`会创建跟epoll空间关联的句柄，`liburing`要求使用

   `struct io_uring ring;`定义一个操作空间，之后的IO操作（包括提交请求SQE，获取结果CQE）都在上面进行，其在用户态通过`mmap`映射了内核中的两个环形队列（提交队列和完成队列），从而实现了内核与用户态的高效通信

   `io_uring_queue_init`负责对该空间进行初始化，第一个参数指定了最多可以有多少个IO任务

   ```c
   /* * entries: 队列深度。指定了最多可以有多少个“在途”未处理的 IO 任务
    * 建议是2的幂，如 4096
    * ring:    指向你定义的结构体指针，库函数会填充它
    * flags:   标志位（如 IORING_SETUP_IOPOLL 开启轮询模式）
    */
   int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags);
   ```

2. sqe：

   类似`epoll`中的`struct epoll_event`，但它们的功能有比较大的区别：

   - `epoll_event` 是告诉内核“我要**监控**这个 fd 的可读事件”
   - `sqe` 是告诉内核“我要**执行**这个具体的 I/O 操作”。 可以将 `sqe` 想象成一张**“任务工单”**，把要做的操作（读、写、连）填好，交给内核

   申请一个IO请求：

   ```c
   // 从 ring 中申请一个空的工单槽位
   struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
   ```

   关于`io_uring_sqe`：通常使用 `io_uring_prep_*` 辅助函数（可以是`accept`，`send`，`recv`，`read`，`write`）来填充，但理解其内部字段有助于理解底层原理

   ```c
   struct io_uring_sqe {
       __u8    opcode;       // 操作码。决定是 Read, Write, Accept 还是 Connect
                             // 类似 epoll_ctl 中的 EPOLL_CTL_ADD 等动作
       __u8    flags;        // 标志位 (如 IOSQE_IO_LINK 开启链式调用)
       __u16   ioprio;       // 优先级
       __s32   fd;           // 要操作的文件描述符 (类似 epoll_ctl 的第一个参数)
       __u64   off;          // 文件的偏移量 (pread/pwrite 用)
       __u64   addr;         // 用户态缓冲区 buffer 的地址 (读写数据的存放地)
       __u32   len;          // buffer 的长度
       
       /* * 【核心字段】user_data 
        * 这是一个 64 位的“回执编号”或“上下文指针”。
        * 类似 epoll_event.data.ptr。
        * 你在这里填什么，内核在完成任务后，会原封不动地在 cqe 里还给你。
        * 通常用来存放用户自定义信息结构体的指针。
        */
       __u64   user_data;    
       
       // ... 其他联合体字段用于特定操作
   };
   ```

3. cqe：类似于 `epoll_wait` 返回的**就绪事件**

   - 在 `epoll` 中，返回意味着“你可以去读写了”（Reactor 模型）
   - 在 `io_uring` 中，返回意味着“**内核已经帮你读写完了**，这是结果”（Proactor 模型），可以将 `cqe` 想象成任务完成后的**“结果回执单”**

   获取完成的IO请求：

   ```c
   struct io_uring_cqe *cqe;
   // 阻塞等待至少一个任务完成
   io_uring_wait_cqe(&ring, &cqe);
   // 或者非阻塞查看
   io_uring_peek_cqe(&ring, &cqe);
   ```

   关于`io_uring_cqe`：结构非常精简，只包含结果必须的信息

   ```c
   struct io_uring_cqe {
       /* * 【核心字段】user_data
        * 对应 sqe 中填入的 user_data。
        * 通过它，你才能知道这个结果是属于哪一个 fd 的，或者是哪一次 read 操作的。
        */
       __u64   user_data;    
   
       /* * 【核心字段】res
        * 操作的返回值。
        * 如果 >= 0：表示成功传输的字节数 (类似 read() 的返回值)。
        * 如果 < 0：表示出错，值为 -errno (如 -EAGAIN, -ECANCELED)。
        */
       __s32   res;          
   
       /* * flags
        * 元数据标志。例如在使用 Provided Buffers 时，这里会包含
        * 内核自动选用的 Buffer ID。
        */
       __u32   flags;        
   };
   ```

## engine

engine是协程执行单元，负责存储所有的task和异步IO并向外提供执行的接口

要存储任务，就要有对应的存储空间，这里选择使用[atomic queue](https://github.com/max0x7ba/atomic_queue)无锁队列来存储对应的协程任务，在这里不存储具体的task对象，然后存储`std::coroutine_handle<>`

### 初始化

1. engine成员：

   ```c++
   // 封装操作后的io_uring
   uring_proxy m_upxy;
   
   // 无锁队列存储协程
   mpmc_queue<coroutine_handle<>> m_task_queue;
   
   atomic<size_t> m_submit_io{0}; // 当前待提交的io操作数量
   atomic<size_t> m_running_io{0}; // 当前正在执行的io操作数量
   ```

2. 线程局部变量

   为简单处理多线程的情况，一旦某个协程句柄被提交到某个engine的任务队列中，即使其暂停恢复后，也应该继续在原先的engine上执行，所以设置一个线程局部变量：

   ```c++
   struct local_info
   {
       context* ctx{nullptr};
       engine*  egn{nullptr};
   };
   inline thread_local local_info linfo;
   ```

3. 初始化和销毁：

   ```c++
   auto engine::init() noexcept -> void
   {
       // 设置线程局部变量
       linfo.egn = this;
       m_upxy.init(config::kEntryLength);
       m_submit_io.store(0, memory_order_relaxed);
       m_running_io.store(0, memory_order_relaxed);
   }
   
   auto engine::deinit() noexcept -> void
   {
       linfo.egn = nullptr;
       m_upxy.deinit();
       mpmc_queue<coroutine_handle<>> empty_queue;
       m_task_queue.swap(empty_queue); // 清空任务队列
   }
   ```

### 添加IO任务

当协程中需要执行IO任务时，分两步：

1. 从engine中获取一个sqe
2. 告诉engine当前提交了一个IO操作，对应的`m_submit_io`就要加1

```c++
auto engine::get_free_urs() noexcept -> ursptr
{
    return m_upxy.get_free_sqe();
}
auto engine::add_io_submit() noexcept -> void
{
    m_submit_io.fetch_add(1, memory_order_relaxed);
    wakeup(); // 后面解释作用
}
```

### 提交并执行协程任务

1. 提供提交任务的接口，外部可以通过对应的接口来提交对应的协程句柄：

   ```c++
   auto engine::submit_task(coroutine_handle<> handle) noexcept -> void
   {
       if (!m_task_queue.try_push(handle)) {
           wakeup();
       } else {
           // 队列满, 应该直接运行协程或者丢掉该协程并报错
       }
   }
   ```

2. 执行协程任务：

   - **`ready()`** ：工作线程用来判断 engine 的任务队列是否还有待执行的任务
   - **`num_task_schedule()`** ：得到当前 engine 的任务队列还有多少任务
   - **`exec_one_task()`** ：调用`schedule()`，engine 会从任务队列中弹出一个任务并作为返回值返回，然后恢复对应协程的执行
   - **`empty_io()`** ：工作线程调用此函数来判断 engine 内部是否还有未处理的 I/O 任务，这包括未提交和正在执行但未完成的 I/O，如果`empty_io()`返回了 false 那么即使工作线程收到停止信号也不会直接停止，因为这表明还有任务没有执行完成。
   - **`poll_submit()`** ：这是 engine 最为核心的函数，实现对 I/O 的提交以及处理已经完成的 I/O，对于从 io_uring 取出的 cqe，采取批量消费的模式，使用`handle_cqe_entry()`函数，获取IO的结果，如果有回调，就执行对应的回调函数。而在处理 I/O 的过程中也应该变更 I/O 执行状态

   ```c++
   auto engine::ready() noexcept -> bool
   {
       return !m_task_queue.was_empty();
   }
   
   auto engine::num_task_schedule() noexcept -> size_t
   {
       return m_task_queue.was_size();
   }
   
   auto engine::schedule() noexcept -> coroutine_handle<>
   {
       auto handle = m_task_queue.pop();
       return handle;
   }
   
   auto engine::exec_one_task() noexcept -> void
   {
       auto coro = schedule();
       coro.resume();
   }
   
   auto engine::empty_io() noexcept -> bool
   {
       return m_submit_io == 0 && m_running_io == 0;
   }
   
   auto engine::poll_submit() noexcept -> void
   {
       // 当前有可提交的IO任务，就提交io_uring中
       if (m_submit_io > 0) {
           m_upxy.submit();
           m_running_io += m_submit_io;
           m_submit_io.store(0, memory_order_relaxed);
       }
       // 等待任务完成, peek_uring()确保当前有IO任务完成才会消费, 否则当前engine是由于其他原因被唤醒, 跳过处理IO的部分
       if (m_upxy.wait_eventfd() && m_upxy.peek_uring()) {
           m_upxy.handle_for_each_cqe(
               std::bind(&engine::handle_cqe_entry, this, std::placeholders::_1),
               true
           );
       }
   }
   ```

### wakeup

当engine执行了协程任务，并提交了IO任务时，从其角度来看，当前没有协程任务了，也没有IO任务要提交了，需要做的事情就是等待新的协程或者IO任务到来，或者IO任务完成的通知

这一点通过io_uring与eventfd的配合来解决

eventfd 是Linux下的轻量级的用于事件通知的文件描述符，可以在io_uring实例中注册一个eventfd，那么每当有IO完成时，io_uring就会向其写入一个值

如果eventfd中没有数据，那么从eventfd中读数据本身是一个阻塞的操作，可以利用这一点，在`poll_submit()`中阻塞当前的工作线程，等待IO任务完成

如果有新的协程任务或者IO任务到来，可以利用这一机制，主动向eventfd中写入一个值，`poll_submit()`检查当前不是由于IO任务完成而被唤醒，就会跳过消费cqe这一行为，转而去执行新的协程任务/提交IO任务

## context

engine向外提供了提交task和异步IO操作的接口，那么context需要利用这些接口，去执行协程任务或是提交IO操作，具体做法是开启一个线程来进行事件循环：

1. 执行计算任务（`exec_one_task`）
2. 提交IO任务并等待（`poll_submit`）
3. 判断当前是否已经执行完所有任务，还有就阻塞，没有就退出

理想的情况是外部定义一个context，提交要执行的协程任务（要提交IO任务可以开启一个协程任务，在里面执行IO任务，原因是IO任务本身被设计成awaiter而不是task），然后调用run函数启动事件循环，等待任务执行完毕后自动退出即可

### 初始化

context持有一个engine对象和一个工作线程，使用工作线程是为了外部在使用context时无需关心context内部的实现（包括初始化和清理工作）

1. 成员变量

   ```c++
   engine   m_engine; // 执行引擎，提供执行函数的接口
   unique_ptr<jthread> m_job; // jthread, 能够在析构时自动join, 也能接收停止信号
   ctx_id              m_id;
   
   atomic<int> m_waited_cnt{0}; // 记录当前有多少陷入suspend状态的协程任务
   std::function<void()> m_try_stop{nullptr}; // 停止context的运行
   ```

2. 初始化和清理：

   ```c++
   auto context::init() noexcept -> void
   {
       // 线程局部变量记录当前的context
       linfo.ctx = this;
       // 初始化执行引擎
       m_engine.init();
       m_waited_cnt = 0;
   }
   
   auto context::deinit() noexcept -> void
   {
       linfo.ctx = nullptr;
       m_engine.deinit();
       m_waited_cnt = 0;
   }
   ```

### 事件循环

三个动作：

1. 执行计算任务（`exec_one_task`）
2. 提交IO任务并等待（`poll_submit`）
3. 判断当前是否已经执行完所有任务，还有就阻塞，没有就退出

```c++
auto context::run(stop_token token) noexcept -> void
{
    while (!token.stop_requested()) {
        int task_num = m_engine.num_task_schedule();
        for (int i = 0; i < task_num; i++) {
            m_engine.exec_one_task();
        }
    
        if (check_stop_sig()) {
            if (m_try_stop) {
                m_try_stop();
            }
        }
        m_engine.poll_submit();
    }
}
```

判断条件发生在提交IO任务之前，这是因为提交IO任务是一个阻塞函数，设想一种情况，外部通过`submit`接口预先提交了所有任务，但是里面没有涉及IO，那么当这一组任务在context中由`exec_one_task`一次性执行完毕后，没有协程任务也没有IO任务了，自然应该退出了，如果先执行`poll_submit`，会阻塞在eventfd的读取上，外部没有提交协程任务或者IO任务，内部也没有IO任务完成，就会一直阻塞下去

#### 判断当前是否能够停止

三个条件：

1. 有没有待运行的协程任务
2. 有没有处于`suspend`状态的协程任务
3. 有没有未完成/未提交的IO任务

```c++
auto context::check_stop_sig() noexcept -> bool {
    return m_engine.empty_io() && m_waited_cnt <= 0 && !m_engine.ready();
}
```

#### 发送停止信号

`jthread`可以发送停止信号：

```c++
jthread::request_stop();
```

当`jthread`执行的函数需要循环查看`stop_token`，然后主动退出：

```c++
auto context::run(stop_token token) noexcept -> void
{
    while (!token.stop_requested()) {
    	...
    }
}
```

于是我们可以提前定义一个函数对象`m_try_stop`，当context符合条件时，主动发送线程停止信号，同时向eventfd写入一个值，使后面的`poll_submit`不会阻塞：

```c++
m_try_stop = [this]() {
    notify_stop();
};
// 
auto context::notify_stop() noexcept -> void
{
    m_job->request_stop();
    m_engine.wakeup();
}
```

### start

start函数中运行工作线程，因为`jthread`使用智能指针管理，所以单个context可以实现复用（前提是`jthread`工作线程正常结束工作了）

```c++
auto context::start() noexcept -> void
{
    m_job = make_unique<jthread>(
        [this](stop_token token)
        {
            this->init();
            if (m_try_stop == nullptr) {
                m_try_stop = [this]() {
                    notify_stop();
                };
            }
            this->run(token);
            this->deinit();
        });
}
```

之后就可以使用如下方式来使用context：

```c++
context ctx;
ctx.submit_task(...);
// repeat submit...
ctx.start(); // 启动运行(非阻塞)
ctx.join(); // 主动等待 context 完成所有任务
// 后面要使用context可以继续submit然后start
```

## scheduler

为了充分利用cpu的多核能力，创建scheduler来管理多个context，方便并发执行各个协程或IO任务，用户无需手动管理各个context，只要把任务交给scheduler，由scheduler进行任务的调度

流程：

```c++
scheduler::init(); // 入参为 context 数量，默认为当前机器的逻辑 CPU 核心数
submit_to_scheduler(task);
// more submit...
scheduler::loop(); // 等待全部 context 完成任务后再关闭全部 context 并返回(阻塞式)
```

当调用`scheduler::loop`时，程序会调用context开始执行任务，并阻塞等待所有任务完成

### 任务管理

当调用多个context执行任务的时候，我们不希望每个context在执行完自己的任务后，检查到队列中没有任务了就停止，而是希望由scheduler统一发送停止信号，这一点可以通过设置各个context的`m_try_stop`为一个空函数（非`nullptr`）来做到

其次，既然需要scheduler来统一发送停止信号，scheduler就必须知道什么时候停止各个context，这时候出现了一个矛盾，本来是每个context掌管自己运行任务的数量，scheduler现在要强行接管这一功能，还要同时清楚多个context上的任务数量，看上去并不易实现

接下来介绍的包装协程可以解决这一痛点

#### wrapper_task

lab1中介绍的通用型`task`类可以封装各种类型的协程任务，但有一点不好，在`co_return`后的`final_suspend`会返回给上一个协程任务，没有就暂停自身，这可能影响协程帧的释放：

```c++
task<void> func1() {
    ...
    co_await IO; // 将自身挂起等待IO完成
    co_return;
}
task<void> func2() {
    co_await func1();
    ...
    co_return;
}
```

试想一下，当我们将`func2()`提交给`scheduler`或者`context`，被执行引擎运行时：

```c++
auto engine::exec_one_task() noexcept -> void
{
    auto coro = schedule();
    coro.resume();
}
```

其运行过程应该是：

1. `engine`取出`func2()`的句柄，然后`resume()`执行
2. 执行`func2()`
3. `co_await func1()`将执行权转移给`func1()`
4. 执行`func1()`
5. `co_await IO`，`func1()`会被挂起
6. 执行权回到`engine::exec_one_task()`中，继续执行

当一个IO完成，`func1()`的句柄被重新送入任务队列：

1. `engine`取出`func1()`的句柄，然后`resume()`执行
2. 执行`func1()`
3. `co_return`将执行权转移给`func2()`
4. 执行`func2()`
5. `co_return`将执行权转移给`noop_coroutine`，同时`func1()`对应的`task`被析构，释放对应的句柄
6. `noop_coroutine`什么都不做，执行权回到`engine::exec_one_task()`中继续执行

此时，如果外部没有保存`func2()`的句柄，`func2()`对应的协程帧就永远不会被释放（除非被外部唤醒或是由外部主动`destroy`释放），那么就会出现内存泄露

一个可行的解决办法是，定义一个只允许在`scheduler`或者`context`中的使用的特别`task`类，该类负责对用户传入的协程进行一次包装，并在`final_suspend`时返回`suspend_never`：

```c++
auto make_wrapper_task(coro::task<void> user_task) -> wrapper_task {
    co_await user_task;
    co_return;
    // user_task will be destruct, its coroutine frame will be deleted
}
```

这样，外部提交的协程都会由于RAII，使得协程句柄在`task`的析构函数中释放，而`wrapper_task`则由于`final_suspend`返回`suspend_never`，直接运行到`clean code`清除自身

`wrapper_task`只封装右值`task<void>`类型，这是因为用户提交到`scheduler`或`context`的协程任务不应该在意返回值，同时也不应该在外部持有协程句柄造成二次释放，如果协程有返回值需要处理，应该提前封装：

```c++
task<int> inner() {
    ...
    co_return 1;
}
task<void> to_submit() {
    auto n = co_await inner();
    co_return;
}
submit_to_scheduler(to_submit());
// 或者如下方式:
// auto task = to_submit();
// submit_to_scheduler(std::move(task));
```

具体实现：

```c++
// hpp
class wrapper_task;

class wrapper_promise {
public:
    wrapper_promise();
    ~wrapper_promise();
    // delete the copy constructor and operator
    wrapper_promise(const wrapper_promise&) = delete;
    wrapper_promise(wrapper_promise&&) = delete;
    auto operator=(const wrapper_promise&) -> wrapper_promise& = delete;
    auto operator=(wrapper_promise&&) -> wrapper_promise& = delete;
    // promise function
    auto get_return_object() noexcept -> wrapper_task;
    auto initial_suspend() noexcept -> std::suspend_always;
    auto final_suspend() noexcept -> std::suspend_never;
    auto return_void() noexcept -> void;
    auto unhandled_exception() noexcept -> void;
    auto set_before_exit_func(std::function<void()> before_exit_func) noexcept -> void;
private:
    std::function<void()> m_before_exit_func{nullptr};
    std::exception_ptr m_exception_ptr{nullptr};
};
/*
 * this class should be used to encapsulate the top-level 
 * coroutine submitted by the user.
 */
class wrapper_task {
public:
    using promise_type = wrapper_promise;
    
    explicit wrapper_task(std::coroutine_handle<wrapper_promise> handle);
    ~wrapper_task();
    // delete the copy constructor and operator
    wrapper_task(const wrapper_task&) = delete;
    wrapper_task(wrapper_task&&);
    auto operator=(const wrapper_task&) -> wrapper_task& = delete;
    auto operator=(wrapper_task&&) ->wrapper_task&;
    // nodiscard指明编译器去检查返回值有没有被使用过, 即强制调用者去使用返回值
    [[nodiscard]] auto promise() const -> const wrapper_promise& {
        return m_handle.promise();
    }
    [[nodiscard]] auto promise() -> wrapper_promise& {
        return m_handle.promise();
    }
    [[nodiscard]] auto handle() const -> const std::coroutine_handle<wrapper_promise>& {
        return m_handle;
    }
    [[nodiscard]] auto handle() -> std::coroutine_handle<wrapper_promise>& {
        return m_handle;
    }
    auto resume() -> bool {
        if (!m_handle.done()) {
        m_handle.resume();
    }
    return !m_handle.done();
    }
private:
    std::coroutine_handle<wrapper_promise> m_handle{nullptr};
};

auto make_wrapper_task(coro::task<void> user_task) -> wrapper_task;

// cpp
wrapper_promise::wrapper_promise() {

}

wrapper_promise::~wrapper_promise() {

}

auto wrapper_promise::get_return_object() noexcept -> wrapper_task {
    return wrapper_task{std::coroutine_handle<wrapper_promise>::from_promise(*this)};
}

auto wrapper_promise::initial_suspend() noexcept -> std::suspend_always {
    return std::suspend_always{};
}

auto wrapper_promise::final_suspend() noexcept -> std::suspend_never {
    if (m_before_exit_func) {
        m_before_exit_func();
    }
    return std::suspend_never{};
}

auto wrapper_promise::return_void() noexcept -> void {

}
auto wrapper_promise::unhandled_exception() noexcept -> void {
    // store it, but didn't use it, user can not touch this
    m_exception_ptr = std::current_exception();
}
auto wrapper_promise::set_before_exit_func(std::function<void()> before_exit_func) noexcept -> void {
    m_before_exit_func = std::move(before_exit_func);
}

wrapper_task::wrapper_task(std::coroutine_handle<wrapper_promise> handle)
    : m_handle(handle) {

}
wrapper_task::~wrapper_task() {
    // no-op
    // suspend_never means once co_return, coroutine frame will self-destruct
}
wrapper_task::wrapper_task(wrapper_task&& other) :m_handle(std::exchange(other.m_handle, nullptr)) {

}
auto wrapper_task::operator=(wrapper_task&& other) -> wrapper_task& {
    if (std::addressof(other) != this) {
        if (m_handle != nullptr) {
            m_handle.destroy();
            m_handle = nullptr;
        }
        m_handle = std::exchange(other.m_handle, nullptr);
    }
    return *this;
}

auto make_wrapper_task(coro::task<void> user_task) -> wrapper_task {
    co_await user_task;
    co_return;
    // user_task will be destruct, its coroutine frame will be deleted
}
```

#### 任务计数

从`submit_to_scheduler()`提交的协程默认被视为顶层协程，只有其生命周期结束，才意味着一个协程任务的结束，这使我们可以不再关心协程内部的运行情况，因此可以设置一个原子计数器，每当`submit_to_scheduler()`被调用时，将计数器加1，当顶层协程结束时，借由`wrapper_task`，将计数器减1，当计数器为0时，代表着所有协程任务已经被执行完毕，可以退出

### 初始化

scheduler采用单例模式来实现，为了外部使用方便，只对外保留静态接口，将`getInstance`设置为私有

1. 变量

   ```c++
   // 静态变量
   private:
   	// 外部提交的顶层协程个数
       static std::atomic<int> m_task_cnt;
   	// 条件变量, 负责在协程任务结束后唤醒阻塞的线程
       static std::condition_variable m_cv;
   	static std::mutex m_mux;
   // 成员变量
   private:
   	// context个数
       size_t                                              m_ctx_cnt{0};
       // 存储context
   	detail::ctx_container                               m_ctxs;
       // 派发机制
   	detail::dispatcher<coro::config::kDispatchStrategy> m_dispatcher;
   ```

2. 懒汉单例：

   ```c++
   private:
   static auto get_instance() noexcept -> scheduler*
   {
       static scheduler sc;
       return &sc;
   }
   ```

3. 初始化变量

   ```c++
   inline static auto init(size_t ctx_cnt = std::thread::hardware_concurrency()) noexcept
       -> void
   {
       if (ctx_cnt == 0)
       {
           ctx_cnt = std::thread::hardware_concurrency();
       }
       get_instance()->init_impl(ctx_cnt);
   }
   // .cpp
   // 静态变量在cpp文件中初始化
   std::atomic<int> scheduler::m_task_cnt = 0;
   std::condition_variable scheduler::m_cv;
   std::mutex scheduler::m_mux;
   auto scheduler::init_impl(size_t ctx_cnt) noexcept -> void
   {
       detail::init_meta_info();
       m_task_cnt = 0;
       m_ctx_cnt = ctx_cnt;
       m_ctxs    = detail::ctx_container{};
       m_ctxs.reserve(m_ctx_cnt);
       for (int i = 0; i < m_ctx_cnt; i++)
       {
           m_ctxs.emplace_back(std::make_unique<context>());
       }
       m_dispatcher.init(m_ctx_cnt, &m_ctxs);
   }
   ```

### 提交任务

对于外部提交进来的任务，使用`wrapper_task`封装，增加原子计数，在`wrapper_task`的`final_suspend`函数中减少原子计数：

```c++
inline void submit_to_scheduler(task<void>&& task) noexcept
{
    scheduler::submit(std::move(task));
}
// 类函数
static inline auto submit(task<void>&& task) noexcept -> void
{
    m_task_cnt.fetch_add(1, std::memory_order_acq_rel);
    auto wrapper_task = detail::make_wrapper_task(std::move(task));
    // 设置原子计数的增减
    wrapper_task.promise().set_before_exit_func([]() {
        m_task_cnt.fetch_sub(1, std::memory_order_acq_rel);
        if(m_task_cnt.load(std::memory_order_relaxed) == 0) {
            m_cv.notify_all();
        }
    });
    auto handle = wrapper_task.handle();
    submit(handle);
}
auto scheduler::submit_task_impl(std::coroutine_handle<> handle) noexcept -> void
{
    // 获取当前要派发到context的对应id
    size_t ctx_id = m_dispatcher.dispatch();
    m_ctxs[ctx_id]->submit_task(handle);
}
```

### 执行任务

在`loop`函数中开启所有的context运行，利用条件变量阻塞当前线程，等待所有任务完成：

```c++
auto scheduler::loop_impl() noexcept -> void
{
    for (int i = 0; i < m_ctx_cnt; i++) {
        m_ctxs[i]->set_try_stop([]() {
            // 设置空函数体, context不会自己停止
            return;
        });
    }
    for (int i = 0; i < m_ctx_cnt; i++) {
        m_ctxs[i]->start();
    }
    {
        std::unique_lock<std::mutex> lock(m_mux);
        m_cv.wait(lock, [this]() {
            return m_task_cnt.load(std::memory_order_relaxed) == 0;
        });
    }
    // 所有任务完成, 统一发送停止信号
    stop_impl();
}
```

所有任务完成后，告知所有context，任务结束：

```c++
auto scheduler::stop_impl() noexcept -> void
{
    for (int i = 0; i < m_ctx_cnt; i++)
    {
        m_ctxs[i]->notify_stop();
    }
    for (int i = 0; i < m_ctx_cnt; i++) {
        m_ctxs[i]->join();
    }
}
```

