---
title: c++20制作简易协程库5
publishDate: 2025-11-14 22:00:00
description: ''
tags:
  - modern cpp
  - coroutine
  - tutorial
language: '中文'
---

## 协程高级同步组件

高级同步组件，这里指的是需要配合的各种同步组件，比如：

条件变量condition_variable的实现要和互斥锁mutex配合

通道channel的实现要使用互斥锁mutex和条件变量condition_variable共同实现

### condition_variable

协程意义下的条件变量，与`std::condition_variable`类似，主要api为`wait`和`notify()`，当在协程中调用`wait()`函数时，应该让当前协程陷入suspend状态，而线程可以继续运行

```c++
auto wait(mutex& mtx) noexcept -> awaiter; // 协程调用，awaiter 的 await_resume 返回 void
auto wait(mutex& mtx, cond_type&& cond) noexcept -> awaiter; // 协程调用，awaiter 的 await_resume 返回 void
auto wait(mutex& mtx, cond_type& cond) noexcept -> awaiter; // 协程调用，awaiter 的 await_resume 返回 void
auto notify_one() noexcept -> void; // 普通调用
auto notify_all() noexcept -> void; // 普通调用
```

- `wait`： 阻塞当前协程，直到另一个协程调用同一个`condition_variable`实例的`notify`系方法，或者直到指定的谓词函数 (即条件) 返回 true。如果没有收到通知，即使条件为 true 也不会继续执行。如果收到通知了，但是条件不成立将仍然被阻塞，不会执行
- `notify_one`：唤醒一个等待该条件变量的协程
- `notify_all`：唤醒所有等待该条件变量的线程

条件变量通常需要和互斥量mutex共同使用：

1. `co_await cv.wait(mutex)`：将获得的锁释放，将协程加入条件变量的等待队列，等待被唤醒，然后获取锁，继续执行

2. `co_await cv.wait(mutex, pred)`：语义即：

   ```c++
   while (!pred()) {
       co_await wait(mutex);
   }
   ```

3. `notify_xxx`：唤醒条件变量的等待队列中一个协程

从上面的介绍可以看出，在使用wait系列的函数之前， 必须先获得mutex锁：

```c++
condition_variable cv;
mutex mtx;
int global_id;
task<> func(int id) {
  auto guard = co_await mtx.lock_guard();
  co_await cv.wait(mtx, [&](){id==global_id;});
  global_id+=1;
  cv.notify_all();
}
```

**实现里特别需要注意的点在于**：condition_variable本身会维护一个等待队列，但如果其被唤醒了，需要等锁，那么该协程应该转移到mutex的等待队列中，不再由condition_variable的等待队列负责

同时，notify系列的函数，可以在不获取互斥锁的情况下被调用，也就是说，notify是可以被并行执行的，那就需要保证状态的线程安全性，因此，设计状态时，无法像mutex一样，只使用一个原子变量就能保证协程的等待与恢复是以**FIFO**实现的，综合性能和实用性来考虑，这里使用自旋锁**spinlock**和**队列头尾指针**来作为状态（使用锁的逻辑比直接使用原子变量的逻辑更加清晰）

1. 状态设计：

   ```c++
   detail::spinlock m_spin;	// 自旋锁
   cv_awaiter* m_awaiter_head{nullptr}; // 队列头指针
   cv_awaiter* m_awaiter_tail{nullptr}; // 队列尾指针
   ```

2. 状态变化：

   安全的含义是使用自旋锁来保证头尾指针读写的线程安全

   - `wait`：其状态变化很简单，就是安全的将awaiter指针加入条件变量的等待队列，然后**释放协程锁**
   - 带条件的`wait`：先检查条件是否为true，如果为true了，那么不应该等待，直接恢复这个协程，否则像wait一样，安全的将awaiter指针加入条件变量的等待队列，然后**释放协程锁**
   - `notify_one`：从条件变量的等待队列中取出第一个awaiter指针，**尝试让它获取锁**（即让它尝试加入协程锁的等待队列中），如果成功加入了协程锁的等待队列，那么该awaiter携带的协程将由协程锁mutex的`unlock`负责恢复，否则（没有加入协程锁的等待队列，代表获取了锁），直接恢复
   - `notify_all`：首先安全的替换条件变量的队列头指针，然后一次性对所有awaiter指针做**尝试获取锁**的操作即可

关于**尝试让condition_variable的awaiter获取锁**的操作，可以选择将其设置为mutex的的友元类，然后通过复现mutex的awaiter对mutex的状态来将其加入mutex的等待队列

另一种方式是，令condition_variable的awaiter继承自mutex的awaiter，自动复用其中的代码

#### cv_awaiter的实现

```c++
struct cv_awaiter : public mutex::mutex_awaiter {
    condition_variable& cv;
    cond_type pred;
    using mutex_awaiter::mutex_awaiter;
    cv_awaiter(mutex& mutex, condition_variable& condition_variable): 
    mutex::mutex_awaiter(mutex), cv(condition_variable), pred(nullptr) {}

    cv_awaiter(mutex& mutex, condition_variable& condition_variable, cond_type& cond): 
    mutex::mutex_awaiter(mutex), cv(condition_variable), pred(cond) {}

    cv_awaiter(mutex& mutex, condition_variable& condition_variable, cond_type&& cond): 
    mutex::mutex_awaiter(mutex), cv(condition_variable), pred(std::move(cond)) {}

    ~cv_awaiter() override {}

    auto await_ready() noexcept -> bool {
        ctx.register_wait();
        return false;
    }

    auto await_suspend(std::coroutine_handle<> awaiting_handle) noexcept -> bool {
        handle = awaiting_handle;
        return add_waiting_list();
    }
    auto await_resume() noexcept -> void {
        ctx.unregister_wait();
    }
	// cv_awaiter自己的add_waiting_list
    auto add_waiting_list() noexcept -> bool {
        if (!pred || pred() == false) {
            // 挂起
            next = nullptr;
            {
                std::lock_guard<detail::spinlock> lock(cv.m_spin);
                if (cv.m_awaiter_head == nullptr) {
                    cv.m_awaiter_head = cv.m_awaiter_tail = this;
                } else {
                    cv.m_awaiter_tail->next = this;
                    cv.m_awaiter_tail = this;
                }
            }
            // 解锁
            mtx.unlock();
            return true;
        }
        // 不挂起, 继续执行
        return false;
    }
    // 尝试让condition_variable的awaiter获取锁
    auto try_wake() noexcept -> void {
        // 复用mutex_awaiter的add_waiting_list
        if (!mutex_awaiter::add_waiting_list()) {
            resume_coro();
        }
        // 获取失败, this会被挂载到mtx所在的链表上, 由mtx.unlock调用重载后的resume_coro
    }
    auto resume_coro() noexcept -> void override {
        // resume_coro对应的handle获得了协程锁mtx
        if (!pred || pred() == false) {
            // 虚假唤醒, 加回链表并释放锁
            next = nullptr;
            {
                std::lock_guard<detail::spinlock> lock(cv.m_spin);
                if (cv.m_awaiter_head == nullptr) {
                    cv.m_awaiter_head = cv.m_awaiter_tail = this;
                } else {
                    cv.m_awaiter_tail->next = this;
                    cv.m_awaiter_tail = this;
                }
            }
            mtx.unlock();
        } else {
            // 获取协程的mtx继续运行
            ctx.submit_task(handle);
        }
    }
};
```

#### wait和notify系列函数

```c++
auto wait(mutex& mtx) noexcept -> cv_awaiter { 
        return cv_awaiter(mtx, *this);
    }
    // 如何避免虚假唤醒?
    auto wait(mutex& mtx, cond_type&& cond) noexcept -> cv_awaiter {
        return cv_awaiter(mtx, *this, std::move(cond)); 
    }

    auto wait(mutex& mtx, cond_type& cond) noexcept -> cv_awaiter {
        return cv_awaiter(mtx, *this, cond); 
    }

    auto notify_one() noexcept -> void {
        cv_awaiter* awaiter = nullptr;
        {
            std::lock_guard<detail::spinlock> lock(m_spin);
            if (m_awaiter_head == nullptr) {
                return;
            }
            awaiter = m_awaiter_head;
            m_awaiter_head = static_cast<cv_awaiter*>(m_awaiter_head->next);
        }
        awaiter->try_wake();
    };

    auto notify_all() noexcept -> void {
        cv_awaiter* awaiter = nullptr;
        {
            std::lock_guard<detail::spinlock> lock(m_spin);
            if (m_awaiter_head == nullptr) {
                return;
            }
            awaiter = m_awaiter_head;
            m_awaiter_head = nullptr;
        }
        while (awaiter) {
            cv_awaiter* temp = static_cast<cv_awaiter*>(awaiter->next);
            awaiter->try_wake();
            awaiter = temp;
        }
    };
```



### channel

channel是一个使用协程锁mutex和协程条件变量的多生产者多消费者的阻塞队列

