---
title: c++20制作简易协程库3
publishDate: 2025-11-14 22:00:00
description: ''
tags:
  - modern cpp
  - coroutine
  - tutorial
language: '中文'
---

## io_info

lab2中提到过sqe数据结构中可以传入一个数据指针，该指针会cqe中原封不动的返回，如果我们想要设置提交的IO类型，与IO绑定的协程句柄等等，使用该数据指针绑定一个结构体就会很方便：

```c++
// 执行的IO类型
enum io_type
{
    nop,
    tcp_accept,
    tcp_connect,
    tcp_read,
    tcp_write,
    tcp_close,
    stdin,
    timer,
    none
};

struct io_info
{
    coroutine_handle<> handle; // IO 绑定的协程句柄
    int32_t            result; // IO 执行完的结果
    io_type            type; // IO 类型
    uintptr_t          data; // IO 绑定的内存区域
    cb_type            cb; // IO 绑定的回调函数
};
```

## io_awaiter

要想在协程中发起一个异步IO操作，需要使用`co_await`关键字，其后跟一个`awaiter`或`awaitable`对象

这里设置对应的`io_await`，实现内部的`ready`，`suspend`，`resume`逻辑，完成基础IO框架：

```c++
class base_io_awaiter
{
public:
    base_io_awaiter() noexcept : m_urs(coro::detail::local_engine().get_free_urs())
    {
        assert(m_urs != nullptr && "io submit rate is too high");
    }

    constexpr auto await_ready() noexcept -> bool { return false; }

    auto await_suspend(std::coroutine_handle<> handle) noexcept -> void { m_info.handle = handle; }

    auto await_resume() noexcept -> int32_t { return m_info.result; }

protected:
    io_info             m_info;
    coro::uring::ursptr m_urs;
};
```

`base_io_awaiter`为所有 IO 相关的awaiter提供了一个基类并实现了 awaiter 的全部调度逻辑。当`base_io_awaiter`被构造时会自动从当前线程绑定的 engine 获取一个sqe，当其被`co_await`时`base_io_awaiter`会在`await_suspend`中记录调用协程的句柄并使该协程陷入 suspend 状态。而在 IO 完成后会通过`await_resume`返回 IO 的执行结果。

### tcp_accept_awaiter

考虑服务器要做一个异步连接的动作：

```c++
class tcp_accept_awaiter : public detail::base_io_awaiter
{
public:
    tcp_accept_awaiter(int listenfd, int io_flag = 0, int sqe_flag = 0) noexcept {
        m_info.type = io_type::tcp_accept;
        m_info.cb   = &tcp_accept_awaiter::callback;

        io_uring_sqe_set_flags(m_urs, sqe_flag);
        io_uring_prep_accept(m_urs, listenfd, nullptr, &len, io_flag);
        io_uring_sqe_set_data(m_urs, &m_info); // old uring version need set data after prep
        local_engine().add_io_submit();
    }

    static auto callback(io_info* data, int res) noexcept -> void {
        data->result = res;
    	submit_to_context(data->handle);
    }

private:
    inline static socklen_t len = sizeof(sockaddr_in);
};
```

1. 将`m_info`的IO类型绑定为`accpet`类型，标识这是一个连接动作

2. 绑定IO完成后的回调函数，该回调函数在IO执行完成后，应由engine执行：

   ```c++
   auto engine::handle_cqe_entry(urcptr cqe) noexcept -> void
   {
       auto data = reinterpret_cast<io::detail::io_info*>(io_uring_cqe_get_data(cqe));
       data->cb(data, cqe->res);
       m_running_io--;
   }
   ```

   回调做两件事：

   - 将IO执行的结果（成功，失败或其他信息）存回`io_info`结构体，这样在协程恢复的时候，执行`await_resume()`就可以获取IO执行的结果
   - 将协程再次提交回原context

3. 使用获取的sqe，填充数据，并告知当前线程所绑定的engine，当前有IO任务需要提交执行

剩余的awaiter都大同小异，都使用对应的`io_uring_pre_*`系列函数填充相应数据

如果需要扩展IO类型，只需要在此基础之上，继承`base_io_awaiter`，在io_uring上寻找对应的函数，填充数据即可





