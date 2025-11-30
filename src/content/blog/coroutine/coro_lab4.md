---
title: c++20制作简易协程库4
publishDate: 2025-11-14 22:00:00
description: ''
tags:
  - modern cpp
  - coroutine
  - tutorial
language: '中文'
---

## 协程同步组件

使用scheduler多线程执行协程任务时，仍会面临与多线程任务相同的问题：共同读写某个变量，常用的手段是使用线程同步组件如`mutex`，`condition_variable`来确保多线程下对共享变量操作的正确性

但线程同步组件会阻塞整个线程，如果协程任务使用了线程同步组件，会使整个线程陷入阻塞态，从而线程上其他的协程任务无法继续执行，那么使用协程带来的优势就将不存在

我们期望设计协程相关的同步组件，当一个协程使用对应的同步组件被阻塞时，只将自身陷入到`suspend`状态中，而不会将整个线程阻塞，使得同一个线程上其他能够运行的协程任务仍然能够继续运行，当同步操作完成时，再由同步组件唤醒，重新将协程提交到对应的context任务队列中，恢复原先的运行

### event

event的功能与`std::promise`类似，带有一个返回类型模板参数且默认为空，并且提供`set`以及`wait`两个接口，`set`可以直接调用，但调用`wait`需要是协程的形式`co_await wait()`

event被设定为模板类，方便`set`函数传值

对于模板参数为空的event，功能即传递信号：

```c++
auto set() noexcept -> void; // 普通调用
auto wait() noexcept -> awaiter; // 协程调用，awaiter 的 await_resume 返回 void
```

不为空，传信号的同时传递对应的值：

```c++
template<typename value_type>
auto set(value_type&& value) noexcept -> void; // 普通调用
auto wait() noexcept -> awaiter; // 协程调用，awaiter 的 await_resume 返回 set 设置的值
```

使用方式：

```c++
event<int> ev;
task<> set_func() {
  // codes...
  ev.set(number);
}
task<> wait_func() {
  auto number = co_await ev.wait();
  // codes ...
}
```

#### 锁

由于不同协程可能会运行在不同的线程上，因此event的设计要实现线程安全，使用`mutex`锁可以轻松的做到这一点：

1. 成员变量：

   ```c++
   std::mutex m_mux;
   std::condition_variable m_cv;
   size_t m_waited{0};		// 等待陷入suspend状态的协程
   bool status{false};		// 当前event有没有被set
   std::queue<base_awaiter*> awaiters;	// 保存陷入suspend状态的协程队列
   ```

2. `struct base_awaiter`：

   当一个协程执行`co_await ev.wait();`时，执行调度流程：

   - 如果event已经处于set的状态，协程就不应该陷入`suspend`状态，

   - 否则，协程将自己陷入`suspend`状态，并将当前的awaiter对象放入待恢复的队列中（`awaiters`）

   ```c++
   struct base_awaiter {
       friend class event_base;
       // 构造: 绑定对应的event对象和context对象, 以便后面恢复协程
       base_awaiter(event_base& event) noexcept : m_event(event), m_ctx(local_context()) {}
       auto await_ready() noexcept -> bool {
           m_ctx.unregister_wait();
           // 对event对象的互斥量上锁
           std::lock_guard<std::mutex> lock(m_event.m_mux);
           // 如果event已经被set了, 执行co_await的协程不应该陷入suspend, 而是继续运行
           if (m_event.status) {
               return true;
           }
           // 否则将awaiter添加要恢复队列中
           m_event.awaiters.push(this);
           // m_waited代表当前还有多少个加入恢复队列的协程没有进入suspend状态
           m_event.m_waited += 1;
           return false;
       }
       auto await_suspend(std::coroutine_handle<> handle) noexcept -> bool {
           // 保存要恢复的协程句柄
           m_handle = handle;
           std::unique_lock<std::mutex> lock(m_event.m_mux);
           m_event.m_waited -= 1;
           // 如果m_waited为0, 说明所有加入队列的awaiter都绑定好了要恢复的协程句柄
           // 这样event对象在set的时候就不会恢复一个没有绑定协程句柄的awaiter
           if (m_event.m_waited == 0) {
               m_event.m_cv.notify_all();
           }
           return true;
       }
       auto await_resume() noexcept -> void{
           m_ctx.unregister_wait();
       }
   protected:
       event_base& m_event;	// 当前awaiter绑定的event对象
       context& m_ctx;			// 协程所在的context对象
       std::coroutine_handle<> m_handle{nullptr};	// 保存陷入suspend状态的协程句柄
   };
   ```

3. `class event_base`：

   ```c++
   class event_base {
   public:
       friend struct base_awaiter;
   public:
       event_base() noexcept {};
       ~event_base() noexcept {}
   
   protected:
       // set时, 恢复所有的协程
       void resume_awaiters() noexcept {
           std::queue<base_awaiter*> to_resume;
           {
               std::unique_lock<std::mutex> lock(m_mux);
               m_cv.wait(lock, [this]() {
                   return m_waited == 0;
               });
               // 记录当前event已经被set了
               status = true;
               awaiters.swap(to_resume);
           }
           // 将所有的协程句柄提交回绑定的context
           while (!to_resume.empty()) {
               auto resume_a = to_resume.front();
               to_resume.pop();
               resume_a->m_ctx.submit_task(resume_a->m_handle);
           }
       }
   protected:
       std::mutex m_mux;
       std::condition_variable m_cv;
       size_t m_waited{0};
       bool status{false};
       std::queue<base_awaiter*> awaiters;
   };
   ```

#### 无锁

锁的功能很强大，但涉及到底层系统调用，开销大，同时会锁住整个线程，不利于高并发

因此考虑使用非阻塞式的**原子操作**、**CAS（Compare-and-Swap）**来实现对共享数据的访问和修改，其可以降低系统调用的次数，但是设计上更加复杂，需要考虑线程安全，内存模型等因素

初次接触无锁编程，仅考虑线程安全，不去考虑内存模型（统一使用最严格的内存序）



无锁编程通常使用原子变量来保存共享数据，对于event来说，需要保存的最关键的数据就是awaiter队列（这里可以用链表实现）

那么对于该原子变量来说可能有三个状态：

1. 没有被set过，并且当前链表长度为0（**kUNSETNOAWAITER**）
2. 没有被set过，并且当前链表长度为n（保存指向链表头的指针）
3. 已经被set（**kSET**）

因为要保存指针类型，使用`atomic<void> m_state`来保存状态，其初始状态应为**kUNSETNOAWAITER**

状态变化：

1. `event::set()`：

   该函数会将`m_state`强制转为**kSET**状态，并将`m_state`原先存的状态取出，如果状态是指向链表头的指针，就恢复协程的运行

2. `event::wait()`：

   该函数的功能是负责等待对应的event被set，关键步骤在`await_suspend()`中：

   将自身挂载到链表上，并让协程陷入阻塞态，但直接修改状态时会出现一些竞争状态，即在该函数执行时，可能有以下情况发生：

   - 状态被修改为了**kSET**
   - 其他协程也在执行`awaiter::await_suspend()`，此时两者都想改变状态，将自身添加到链表头部

   对于竞争状态，需要手动进行**循环CAS**，即“如果值没被别人改过，那就更新成功；如果别人先一步改了，我就失败并重试。”

   循环CAS的核心是失败后的重试，这要求成功之前都必须处在一个循环过程中：

   ```c++
   while (true) {
       // 当前已经被set了, 就退出
       if (m_state == kSET) {
           break;
       }
       // 原子变量的CAS操作, 尝试挂载自身到链表中
   	if (try_add_to_list(this, m_state)) {
           // 成功了, 退出循环
           break;
       }
       // 失败了, 循环重试
   }
   ```

   当多个线程上的协程并发执行时，使用上面的操作可以保证线程安全，但缺点也很明显：

   - 需要分析状态，状态越复杂，代码就越复杂
   - 竞争时cpu都处于空转状态（因为一段时间内都在循环重试）

**c++中的原子变量提供了各种CAS操作，常见的有：**

```c++
// expected: 期望的原子变量的状态
// desired: 替换为的状态
// 语义: 如果当前原子变量的状态为expected, 就尝试替换为desired, 替换成功返回true, 否则返回false
// 如果返回false, expected会被改为当前原子变量的新值
bool compare_exchange_strong(T& expected, T desired,
                             memory_order success_order,
                             memory_order failure_order) noexcept;

bool compare_exchange_weak(T& expected, T desired,
                           memory_order success_order,
                           memory_order failure_order) noexcept;
```

其中的`memory_order`是内存序，默认为`memory_order_seq_cst`，也就是保证线程安全的一档，内存序太过复杂，这里不多涉及，使用默认就好

`strong`和`weak`的区别在于`weak`可能会假失败（即使原子变量中存的是`expected`值也有可能替换失败返回false），但是执行速度会快一些，一般在循环中使用`weak`，因为可以容忍少数的假失败来提升速度

**原子变量基本读写操作：**

```c++
// 内存序默认memory_order_seq_cst
// 原子读
T load(memory_order order);
// 原子写
void store(T val, memory_order order);
// 原子交换, 返回原子变量中原先存的值
T exchange(T val, memory_order order);
```

为什么不直接用`exchange()`之类的函数？

考虑这么一种情况：协程1执行到了`set()`，协程2执行到了`await::suspend()`，协程2检查当前状态不为**kSET**，但在执行`exchange()`之前，协程1执行了CAS并把状态修改为了**kSET**，此时协程2的行为应该是退出`await::suspend()`继续运行，但实际上会执行`exchange()`把状态再次修改，就会出现不符合预期的情况

**无锁代码：**

```c++
class event_base {
public:
    struct base_awaiter {
        friend class event_base;
        base_awaiter(event_base& event) noexcept : m_event(event), m_ctx(local_context()) {}
        auto await_ready() noexcept -> bool {
            m_ctx.register_wait();
            return m_event.is_set();
        }
        auto await_suspend(std::coroutine_handle<> handle) noexcept -> bool {
            m_handle = handle;
            void *old_state = m_event.m_state.load(std::memory_order_acquire);
            while (true) {
                // 竞争情况: 准备队列时, event被set了
                if (old_state == reinterpret_cast<void*>(kSet)) {
                    return false;
                }
                // 正常情况: 把自己加入等待者链表
                m_next = (old_state == kUnsetNowaiter) ? nullptr : static_cast<base_awaiter*>(old_state);
                if (m_event.m_state.compare_exchange_strong(
                    old_state, 
                    this,
                    std::memory_order_acq_rel,
                    std::memory_order_relaxed
                )) {
                    return true;
                }
            }
        }
        auto await_resume() noexcept -> void{
            m_ctx.unregister_wait();
        }
    protected:
        event_base& m_event;
        context& m_ctx;
        std::coroutine_handle<> m_handle{nullptr};
        base_awaiter* m_next{nullptr};
    };
    friend struct base_awaiter;
public:
    event_base() noexcept : m_state(kUnsetNowaiter) {};
    ~event_base() noexcept {}

    auto is_set() const noexcept -> bool {
        return m_state.load(std::memory_order_acquire) == reinterpret_cast<void*>(kSet);
    }

    auto reset() -> void {
        void *expected_state = reinterpret_cast<void*>(kSet);
        m_state.compare_exchange_strong(expected_state, kUnsetNowaiter, std::memory_order_acq_rel);
    }

protected:
    void resume_awaiters() noexcept {
        void* old_state = m_state.exchange(reinterpret_cast<void*>(kSet), std::memory_order_acq_rel);
        if (old_state != reinterpret_cast<void*>(kSet)) {
            // 链表挂载等待的协程, 唤醒
            // base_awaiter* awaiter = static_cast<base_awaiter*>(old_state);
            base_awaiter* head = static_cast<base_awaiter*>(old_state);
            while (head != nullptr) {
                auto* next_awaiter = head->m_next;
                head->m_ctx.submit_task(head->m_handle);
                head = next_awaiter;
            }
        }
    }
protected:
    static constexpr void* kUnsetNowaiter = nullptr;
    static constexpr std::uintptr_t kSet = 1;
    std::atomic<void*> m_state;
};
```

### mutex

协程意义上的锁，当一个协程获取锁而陷入阻塞时，只让自己陷入`suspend`状态，而非通常意义上整个线程陷入阻塞状态，此时，context线程仍然会继续从engine取出协程任务并执行

其api如下：

```c++
auto try_lock() noexcept -> bool; // 普通调用
auto lock() noexcept -> awaiter; // 协程调用，awaiter 的 await_resume 返回 void
auto unlock() noexcept -> void; // 普通调用
auto lock_guard() noexcept -> awaiter; // 协程调用，awaiter 的 await_resume 返回 lock_guard
```

- `try_lock`即尝试获取锁，返回值表示是否成功获取锁
- `lock`作为协程调用需要通过`co_await mutex.lock()`的形式调用
- `unlock`即释放锁，并唤醒一个 suspend awaiter（如果存在的话）
- `lock_guard`封装了一系列复合操作，通过`co_await mutex.lock_guard()`的形式调用来获取锁并返回 lock_guard，而 lock_guard 在生命周期结束后会自动释放锁，该函数只需要在awaiter_resume返回对应的lock_guard

首先明确的是，mutex不能像event那样，使用`std::mutex`来实现，毕竟不能尝试使用锁来实现锁，并且二者语义也不同，因此只能考虑无锁版本

有了event的无锁实现经验，会发现mutex与其类似，核心数据结构是一个awaiter队列，这里同样使用`atomic<void> m_state`来表示一个链表，接下来就是考虑它的状态：

1. 当前锁没有被使用（**kUNLOCKED**）
2. 当前锁被使用了，但没有等待者（**kLOCKED_NO_WAITER**）
3. 当前锁被使用了，并且有等待者（**LOCKED_HAVE_WAITER**，保存指向链表头的指针）

初始状态下应为**kUNLOCKED**

状态变化：

1. `lock()`：

   该函数负责对当前互斥量mutex上锁，其核心操作同样在`await_suspend()`中：

   如果当前互斥量没有上锁，就上锁后，让协程继续执行；

   否则将自身挂载到链表上，并让协程陷入阻塞态；

   同样，直接修改状态时会出现一些竞争状态，即在该函数执行时，可能有以下情况发生：

   - 互斥量被释放了，`m_state`变为了**kUNLOCKED**状态
   - 其他协程也在等待互斥量，并尝试将自身挂到链表中

   ```c++
   while (true) {
       // 当前锁没有获取, 尝试获取
       if (m_state == kUNLOCKED) {
           if (m_state.compare_exchange_weak(
                               kUNLOCKED, 
                               kLOCKED_NO_WATIER)) {
               break;
           }
           // 获取锁失败, 被别的协程获取了, 重试
           continue;
       }
       // 原子变量的CAS操作, 尝试挂载自身到链表中
   	if (try_add_to_list(this, m_state)) {
           // 成功了, 退出循环
           break;
       }
       // 失败了, 循环重试
   }
   ```

2. `unlock()`：

   该函数负责在协程中解除锁：

   - `m_state`不能**kUNLOCKED**状态，因为不能解锁一个没有上锁的互斥量

   - 如果`m_state`为**kLOCKED_HAVE_WAITER**状态，就恢复一个协程，该协程会自动获得锁，并且当前链表为空了，应该将其状态转为**kLOCKED_NO_WAITER**状态
   - 如果`m_state`为**kLOCKED_NO_WAITER**状态，那么应该将其状态改为**kUNLOCKED**

   类似`lock()`，使用CAS循环：

   ```c++
   while (true) {
       // 尝试释放锁
       if (state == kLOCKED_NO_WAITER) {
           // 尝试改变状态为kUNLOCKED
           if (m_state.compare_exchange_weak(
                            kLOCKED_NO_WAITER, 
                            kUNLOCKED)) {
               return;
           }
           // 改变失败, 重试
           continue;
       }
       // 有协程尝试解锁未上锁的mutex, 应该报error
       if (state == kUNLOCKED) {
       	ERROR;
   	}
       // 当前有等待者, 尝试唤醒一个等待者, 并尝试修改状态为 m_state.next 或者kLOCKED_NO_WAITER
      	if (m_state.compare_exchange_weak()) {
           resume();
       }
   }
   ```

   **优化：**

   为实现方便，在**kLOCKED_HAVE_WAITER**状态下，`m_state`按照后进先出的顺序来恢复陷入`suspend`状态的协程，并且相比event的`set`函数，在`unlock()`函数使用了CAS循环，进一步降低了性能

   考虑到`unlock`只会被上锁的协程来调用，这个时候是不会有多个协程同时`unlock()`的，于是可以设置一个awaiter指针（非原子变量），每次解锁时，如果该指针为空指针，并且当前状态为**kLOCKED_HAVE_WAITER**，就将链表转移给awaiter指针，之后别的协程再次释放锁时，直接从该指针恢复协程即可

   ```c++
   // 有协程尝试解锁未上锁的mutex, 应该报error
   if (m_state == kUNLOCKED) {
       ERROR;
   }
   // 
   if (awaiter_ptr == nullptr) {
       if (m_state == kLOCKED_NO_WAITER) {
           // 尝试替代为初始状态
           if (m_state.compare_exchange_strong(
                   kLOCKED_NO_WAITER,
                   kUNLOCKED)) {
               // 替换成功, 退出
               return;
           }
       }
       // 否则当前状态为kLOCKED_HAVE_WAITER, m_state实际存储的是链表头指针
       void* waiting_list = m_state.exchange(kLOCKED_NO_WATIER, std::memory_order_acq_rel);
       assert(waiting_list != kUNLOCKED && "can't unlock unlocked mutex!");
       awaiter_ptr = static_cast<awaiter*>(waiting_list);
       // 如果希望FIFO, 就翻转链表
       
   }
   // 当前实际状态是kLOCKED_HAVE_WAITER
   // 移动头结点
   awaiter* to_resume = awaiter_ptr;
   awaiter_ptr = to_resume->next;
   // 恢复弹出的协程
   to_resume->resume_coro();
   ```

   为什么这里判断了`m_state == kLOCKED_NO_WAITER`后，之后状态不可能被设置为**kLOCKED_NO_WAITER**？

   原因很简单，最多只有一个协程执行`unlock()`，且当前状态不能为**kUNLOCKED**，这代表`lock()`此时也不可能把状态设置为**kLOCKED_NO_WAITER**

核心成员：

1. 成员变量：

   ```c++
   inline static void* kUNLOCKED = nullptr;
   inline static void* kLOCKED_NO_WATIER = reinterpret_cast<void*>(1);
   std::atomic<void*> m_state;	
   mutex_awaiter* m_awaiter_head{nullptr};
   ```

2. `struct mutex_awaiter`：

   ```c++
   struct mutex_awaiter {
       std::coroutine_handle<> handle;
       mutex& mtx;
       context& ctx;
       mutex_awaiter* next;	// 作用类似链表的next指针
       // 其他函数见代码
   };
   ```

其余函数实现都比较简单，就不再赘述

**代码：**

```c++
class mutex
{
public:
    struct mutex_awaiter {
        std::coroutine_handle<> handle;
        mutex& mtx;
        context& ctx;
        mutex_awaiter* next;

        explicit mutex_awaiter(mutex& mutex) noexcept : mtx(mutex), 
            ctx(local_context()), handle(nullptr), next(nullptr) {}

        virtual ~mutex_awaiter() {}    

        auto await_ready() noexcept -> bool {
            ctx.register_wait(); 
            void* expected = kUNLOCKED;
            return mtx.m_state.compare_exchange_strong(
                expected,
                 kLOCKED_NO_WATIER,
                 std::memory_order_acquire);
        }

        auto await_suspend(std::coroutine_handle<> awaiting_handle) noexcept -> bool {
            handle = awaiting_handle;
            return add_waiting_list();
        }

        auto await_resume() noexcept -> void {
            ctx.unregister_wait();
        }

        auto add_waiting_list() noexcept -> bool {
            void* old_state = mtx.m_state.load();
            while (true) {
                // 竞争情况：在我们准备排队时，锁被释放了。
                // 我们可以再次尝试获取锁，避免不必要的挂起。
                if (old_state == kUNLOCKED) {
                    if (mtx.m_state.compare_exchange_weak(
                            old_state, 
                            kLOCKED_NO_WATIER)) {
                        return false; // 成功获取锁，不要挂起。
                    }
                    // CAS 失败，意味着 old_state 被其他线程改变了，循环会用新值重试。
                    continue;
                }

                // 正常情况：锁被持有，我们需要把自己加入链表。
                // old_state 要么是 kLockedNoWaiters，要么是链表头。
                next = (old_state == kLOCKED_NO_WATIER) ? nullptr : static_cast<mutex_awaiter*>(old_state);

                // release 内存序确保了在我们挂起协程之前的代码，对唤醒我们的那个线程是可见的。
                if (mtx.m_state.compare_exchange_weak(
                        old_state, 
                        this)) {
                    return true; // 成功把自己加入队列，现在可以安全挂起了。
                }
                // CAS 失败，循环会用新的 old_state 值重试。
            }
        }

        virtual auto resume_coro() noexcept -> void {
            ctx.submit_task(handle);
        }
    };

    struct guard_awaiter : mutex_awaiter
    {
        using mutex_awaiter::mutex_awaiter;
        auto await_resume() -> detail::lock_guard<mutex> { 
            ctx.unregister_wait();
            return detail::lock_guard<mutex>(mtx); 
        }
    };
public:
    mutex() noexcept : m_state(kUNLOCKED), m_awaiter_head(nullptr) {}
    ~mutex() noexcept {
        assert(m_state.load() == kUNLOCKED);
    }
    mutex(const mutex&) = delete;
    mutex& operator=(const mutex&) = delete;

    auto try_lock() noexcept -> bool { 
        void* expected = kUNLOCKED;
        return m_state.compare_exchange_strong(
            expected, 
            kLOCKED_NO_WATIER);
    }


    auto lock() noexcept -> mutex_awaiter {
        return mutex_awaiter{*this}; 
    };

    auto unlock() noexcept -> void {
        assert(m_state != kUNLOCKED && "can't unlock unlocked mutex!");
        if (m_awaiter_head == nullptr) {
            // kLOCKED_NO_WAITER
            void* expected = kLOCKED_NO_WATIER;
            if (m_state.compare_exchange_strong(
                expected,
                kUNLOCKED,
                std::memory_order_acq_rel,
                std::memory_order_acq_rel)) {
                    // 当前没有等待者
                    return;
                }
            // HAVE WAITER
            void* waiting_list = m_state.exchange(kLOCKED_NO_WATIER, std::memory_order_acq_rel);
            assert(waiting_list != kUNLOCKED && "can't unlock unlocked mutex!");
            m_awaiter_head = static_cast<mutex_awaiter*>(waiting_list);
            // reverse the list, to implement FIFO
            mutex_awaiter* awaiter = m_awaiter_head;
            m_awaiter_head = nullptr;
            while (awaiter != nullptr) {
                auto next_awaiter = awaiter->next;
                awaiter->next = m_awaiter_head;
                m_awaiter_head = awaiter;
                awaiter = next_awaiter;
            }
        }
        // m_state is LOCKED_HAVE_WAITER
        mutex_awaiter* to_resume = m_awaiter_head;
        m_awaiter_head = to_resume->next;
        to_resume->resume_coro();
    };

    auto lock_guard() noexcept -> guard_awaiter { return guard_awaiter{*this};};
private:
    inline static void* kUNLOCKED = nullptr;
    inline static void* kLOCKED_NO_WATIER = reinterpret_cast<void*>(1);
    std::atomic<void*> m_state;
    mutex_awaiter* m_awaiter_head;
};
```



