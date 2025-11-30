---
title: c++20制作简易协程库1
publishDate: 2025-11-14 22:00:00
description: ''
tags:
  - modern cpp
  - coroutine
  - tutorial
language: '中文'
---

## task类

c++20中的协程要求用户自定义一个`task`类作为返回值

同时为`task`类必须绑定`promise_type`类型

对于`promise_type`类型，可以理解为一个官方要求实现的接口，包含以下函数：

```c++
// 创建任务对象
auto get_return_object() noexcept -> task;
// 协程创建时的调度
auto initial_suspend() noexcept -> std::suspend_always;
// 协程返回时的调度
auto final_suspend() noexcept -> std::suspend_never;
// 协程函数无返回值需使用
auto return_void() noexcept -> void;
// 协程函数有返回值需使用
// auto return_value(T val) noexcept -> void;
// 协程执行过程中出现异常
auto unhandled_exception() noexcept -> void;
```

### promise_base

接口函数针对协程的返回值有两种不同的函数：`return_xxx`

并且这两种函数不能同时存在，所以对于协程有返回值和无返回值（`void`），至少需要构建两种不同的`promise_type`，于是可以将共性部分抽取到`promise_base`类中：

- 协程创建时的调度：`initial_suspend()`
- 协程返回时的调度：`final_suspend()`

c++协程是非对称协程，这意味着协程函数之间存在调用关系，当协程因为调度而暂停时，执行权应返回给调用该协程的一方，这么说比较抽象，以入门案例中的最后一个程序为例：

`main`中执行了`add(1, 2, 1)`，

`add(1, 2, 1)`中又执行了`add(2, 30, 2)`

**期望的返回顺序应该像普通函数调用那样**：

`add(2, 30, 2)`执行完毕，返回`add(1, 2, 1)`

`add(1, 2, 1)`执行完毕，返回`main`

但实际上当时的程序没有按照预想的情况进行执行

协程创建和返回时的调度应该满足以上期望的返回顺序，这就需要改造`promise_base`中的调度函数：

#### initial_suspend

对于一个协程函数来说，不希望其在创建时能够直接执行，而是在`initial_suspend`后暂停执行，这样协程函数会返回对应的`task`对象

因此返回`suspend_always{}`即可

#### final_suspend

该调度点在执行`co_return`后，期望的情况是当某个协程函数返回时，执行权应该交给调度它的协程函数，就像普通函数调用那样，这要求保存协程的调用链，具体做法就是在`promise`类中设置一个`std::coroutine_handle<> caller`变量，当执行`final_suspend`调度时，将执行权交给`caller`即可，这要求自定义`final_suspend()`的返回值`final_awaiter`

**code**：

介绍一下代码中的`std::coroutine_handle<> await_suspend(std::coroutine_handle<promise_type> handle) noexcept`：

在`co_return`时，会进入定义的`final_awaiter`的`ready`-`suspend`-`resume`三项流程

`await_suspend`的入参是当前执行`co_return`的协程句柄，返回值是当前协程句柄的调用者的句柄

如果没有调用者，会返回一个`std::noop_coroutine()`，该协程句柄指向一个**无操作协程**，之后就会按照**栈帧**，回到最初恢复协程的**函数帧**中执行

```c++
struct promise_base
{
    struct final_awaiter {
        bool await_ready() noexcept {return false;}
        template<typename promise_type>
        auto await_suspend(std::coroutine_handle<promise_type> handle) noexcept -> std::coroutine_handle<> {
            auto& promise = handle.promise();
            return promise.caller_ != nullptr ? promise.caller_ : std::noop_coroutine();
        }
        void await_resume() noexcept {}
    };
    friend final_awaiter;

    promise_base() noexcept {}
    ~promise_base() {}

    auto initial_suspend() noexcept {
        return std::suspend_always{}; 
    }

    [[CORO_TEST_USED(lab1)]] auto final_suspend() noexcept -> final_awaiter
    {
        return final_awaiter{};
    }
    void set_caller(std::coroutine_handle<> caller) {
        caller_ = caller;
    }
    
    void set_detached() {
        state_ = task_state::DETACHED;
    }

    bool is_detached() {
        return state_ == task_state::DETACHED;
    }
protected:
    std::coroutine_handle<> caller_{nullptr};
};
```



### 特化promise<>

为了处理协程函数中各种返回类型，需要使用模板类型来保存返回结果，但是函数可能返回`void`

当协程函数返回void时，不应该使用`promise`，因为里面已经定义了`return_value`函数，因此需要特化一个模板，专门处理没有返回值的情况

这个类相对简单，只需要定义一个`void return_void()`函数，并附带上记录异常的功能即可

**code**：

```c++
template<>
struct promise<void> : public promise_base
{
    using task_type        = task<void>;
    using coroutine_handle = std::coroutine_handle<promise<void>>;

    promise() noexcept                  = default;
    promise(const promise&)             = delete;
    promise(promise&& other)            = delete;
    promise& operator=(const promise&)  = delete;
    promise& operator=(promise&& other) = delete;
    ~promise()                          = default;

    auto get_return_object() noexcept -> task_type;

    constexpr auto return_void() noexcept -> void
    {
    }

    auto unhandled_exception() noexcept -> void
    {
        m_exception_ptr = std::current_exception();
    }
	// result函数负责获取协程返回值, 由于task<void>类型没有返回值，这里只需要抛出协程执行过程中的异常
    auto result() -> void
    {
        if (m_exception_ptr)
        {
            std::rethrow_exception(m_exception_ptr);
        }
    }

private:
    std::exception_ptr m_exception_ptr{nullptr};
};
```

### promise

#### container容器

为了保存协程函数的返回值，需要定义一个容器，容器有三种状态：

- `unset_return_value`：一个内部定义的空结构体，仅用于表示“未设置值”的初始状态。
- `store_type`：存储协程的返回值
- `std::exception_ptr`：存储从协程

这里直接将三种状态放在一个`union`中，使用`std::variant`来保存（更安全的`union`结构，这意味着任何时刻变量中只能是三者之一）

```c++
struct unset_return_value
{
    unset_return_value() {}
    unset_return_value(unset_return_value&&)      = delete;
    unset_return_value(const unset_return_value&) = delete;
    auto operator=(unset_return_value&&)          = delete;
    auto operator=(const unset_return_value&)     = delete;
};
static constexpr bool return_type_is_reference = std::is_reference_v<T>;
using stored_type =
    std::conditional_t<return_type_is_reference, std::remove_reference_t<T>*, std::remove_const_t<T>>;
using variant_type = std::variant<unset_return_value, stored_type, std::exception_ptr>;
// 保存用户通过co_return返回的数据
variant_type m_storage{};
```

- `std::is_reference_v`：这是一个编译期常量，用于判断模板参数T是否是一个引用类型
- `std::conditional_t`：这是一个编译期的 `if-else`。它根据第一个布尔参数，从后面两个类型中选择一个作为最终类型

`store_type`表示如果T是一个引用，就将存储对应的指针类型（variant不能直接存储引用），否则存储原始类型（去除const）

#### return_value

当协程函数`co_return val;`时触发，此时应将返回值保存到容器中

```c++
template<typename value_type>
        requires(return_type_is_reference and std::is_constructible_v<T, value_type &&>) or
                (not return_type_is_reference and std::is_constructible_v<stored_type, value_type &&>)
auto return_value(value_type&& value) -> void
{
    if constexpr (return_type_is_reference)
    {
        T ref = static_cast<value_type&&>(value);
        m_storage.template emplace<stored_type>(std::addressof(ref));
    }
    else
    {
        m_storage.template emplace<stored_type>(std::forward<value_type>(value));
    }
}
```

为什么不使用类模板T而是新增一个函数模板value_type？

这里实际上使用了一个万能引用（`value_type&&`）来避免不必要的拷贝，并通过`requires`子句确保`co_return`的表达式可以被用来构造上面定义好的`store_type`类型

根据模板T是否是引用类型（比如定义的`task<int&>`，T就是`int&`），就获取传入的值的地址，再由`m_storage`中的指针来接收，否则利用完美转发，将值传给`m_storage`

#### result

获取协程返回值，`return_value`将返回值存在容器中，但要获取它还需要主动调用`promise`对象的`result`方法，此时：

- `m_storage`是`unset_return_value`，报运行时错误
- `m_storage`是`std::exception_ptr`，重新抛出对应的异常
- `m_storage`是`stored_type`，返回对应的值

```c++
auto result() & -> decltype(auto)
{
    if (std::holds_alternative<stored_type>(m_storage))
    {
        if constexpr (return_type_is_reference)
        {
            return static_cast<T>(*std::get<stored_type>(m_storage));
        }
        else
        {
            return static_cast<const T&>(std::get<stored_type>(m_storage));
        }
    }
    else if (std::holds_alternative<std::exception_ptr>(m_storage))
    {
        std::rethrow_exception(std::get<std::exception_ptr>(m_storage));
    }
    else
    {
        throw std::runtime_error{"The return value was never set, did you execute the coroutine?"};
    }
}
```

### task

与`promise`对应，为了表达返回值，需要定义为模板类，这样在定义协程函数时：

```c++
task<void> func1() {
    ...
}
task<int> func2() {
    ...
}
```

`task`类持有一个协程句柄，指向当前与`task`关联的协程帧

1. 为了让`promise`能够构造`task`对象，其需要定义一个从协程句柄转化而来的有参构造函数

2. 在`task`析构函数中，如果持有的协程句柄仍然有效，应该将其销毁

3. 不应有拷贝构造和拷贝赋值

4. 支持移动构造和移动赋值

5. 封装协程句柄的`resume`和`destroy`函数，对外提供操作协程的函数

6. 提供获取`promise`对象和协程句柄的函数

   ```c++
   template<typename return_type>
   class task
   {
   public:
       using task_type        = task<return_type>;
       using promise_type     = detail::promise<return_type>;
       using coroutine_handle = std::coroutine_handle<promise_type>;
   
       task() noexcept : m_coroutine(nullptr) {}
   
       explicit task(coroutine_handle handle) : m_coroutine(handle) {}
       task(const task&) = delete;
       task(task&& other) noexcept : m_coroutine(std::exchange(other.m_coroutine, nullptr)) {}
   
       ~task()
       {
           if (m_coroutine != nullptr)
           {
               m_coroutine.destroy();
           }
       }
   
       auto operator=(const task&) -> task& = delete;
   
       auto operator=(task&& other) noexcept -> task&
       {
           if (std::addressof(other) != this)
           {
               if (m_coroutine != nullptr)
               {
                   m_coroutine.destroy();
               }
   
               m_coroutine = std::exchange(other.m_coroutine, nullptr);
           }
   
           return *this;
       }
   
       /**
        * @return True if the task is in its final suspend or if the task has been destroyed.
        */
       auto is_ready() const noexcept -> bool { return m_coroutine == nullptr || m_coroutine.done(); }
   
       auto resume() -> bool
       {
           if (!m_coroutine.done())
           {
               m_coroutine.resume();
           }
           return !m_coroutine.done();
       }
   
       auto destroy() -> bool
       {
           if (m_coroutine != nullptr)
           {
               m_coroutine.destroy();
               m_coroutine = nullptr;
               return true;
           }
   
           return false;
       }
   
       auto promise() & -> promise_type& { return m_coroutine.promise(); }
       auto promise() const& -> const promise_type& { return m_coroutine.promise(); }
       auto promise() && -> promise_type&& { return std::move(m_coroutine.promise()); }
   
       auto handle() & -> coroutine_handle { return m_coroutine; }
       auto handle() && -> coroutine_handle { return std::exchange(m_coroutine, nullptr); }
   private:
       coroutine_handle m_coroutine{nullptr};
   };
   ```

7. 最重要的是重载`co_await`运算符，以支持`co_await`语义实现协程嵌套执行和正确的调用返回

   对于`co_await func();`的语义，其执行流程是：

   - `auto task_func = func();`

     触发`promise.initial_suspend()`

   - `co_await task_func;`

     触发`task_func`对象中的`co_await`运算符重载

   所以该重载应该返回一个`awaiter`来实行调度，期望的调度是：将执行权由调用协程交给被调用的协程，在`suspend`中就需要返回被调用协程的句柄

   同时为了在被调用协程返回后正确的回到调用协程的位置，需要设置`promise_base`的`caller`成员变量保存调用协程的句柄，这样在被调用协程执行`final_suspend`时正确的转移执行权

   ```c++
   struct awaitable_base
   {
       awaitable_base(coroutine_handle coroutine) noexcept : m_coroutine(coroutine) {}
   
       auto await_ready() const noexcept -> bool { return !m_coroutine || m_coroutine.done(); }
   
       auto await_suspend(std::coroutine_handle<> awaiting_coroutine) noexcept -> std::coroutine_handle<>
       {
           // TODO[lab1]: Add you codes
           auto& m_promise = m_coroutine.promise();
           m_promise.set_caller(awaiting_coroutine);
           return m_coroutine;
       }
   
       std::coroutine_handle<promise_type> m_coroutine{nullptr};
   };
   
   auto operator co_await() const& noexcept
   {
       struct awaitable : public awaitable_base
       {
   		// 子协程co_return -> 子协程调用result_value/result_void -> 子协程调用final_suspend()
           // -> 执行权转移给父协程 -> 父协程调用await_resume获取返回值
           auto await_resume() -> decltype(auto) { return this->m_coroutine.promise().result(); }
       };
   
       return awaitable{m_coroutine};
   }
   
   auto operator co_await() const&& noexcept
   {
       struct awaitable : public awaitable_base
       {
           auto await_resume() -> decltype(auto) { return std::move(this->m_coroutine.promise()).result(); }
       };
   
       return awaitable{m_coroutine};
   }
   ```

#### 通过promise构造task

即实现`promise`的`get_return_object()`函数，只需要调用`task`的有参构造函数，返回`task`对象即可：

```c++
template<typename return_type>
inline auto promise<return_type>::get_return_object() noexcept -> task<return_type>
{
    return task<return_type>{coroutine_handle::from_promise(*this)};
}

inline auto promise<void>::get_return_object() noexcept -> task<>
{
    return task<>{coroutine_handle::from_promise(*this)};
}
```

