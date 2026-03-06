---
title: 'LSM-lab0-1'
publishDate: '2025-12-07'
updatedDate: '2025-12-07'
description: 'TinyLSM-Lab笔记'
tags:
  - modern c++
  - notes
  - lsm tree
  - database
language: '中文'
draft: true
---

## 一. lab0 环境准备

1. `vscode` + `clang++` + `clangd`：参考[linux下c++环境配置](https://liang-bk.github.io/blog/linux-cpp-environment/)

2. `xmake`：跟着LSM-lab配即可

3. 切换工具链到`clang++-20`：

   在`xmake.lua`最上面加上：

   ```lua
   set_toolchains("clang-20")
   ```

4. `xmake`项目使用`codelldb`调试：

   设置→`Xmake: Debug Config Type`

   选择`codelldb`即可

   之后在左侧`xmake`插件或者底部`xmake`工具栏中切换运行目标进行运行和调试

## 二. lab1 SkipList实现

### 2.1 基本结构

一开始以为跳表像book里面画的那样，每层存储结点并有一个向下的指针，实现里发现不是这样，本质是一个双链表：

图

所有的结点数据只保留一份

所谓层实际上是单个结点存储不同步长的`next`指针，以此达到分层的效果

实际代码中存储的是双链表，主要为了方便查找，基本的跳表节点如下（一个结点存一个kv）：

```c++
struct SkipListNode {
  std::string key_;   // kv存储的k
  std::string value_; // kv存储的v
  uint64_t tranc_id_; // 事务 id, 目前可以忽略
  std::vector<std::shared_ptr<SkipListNode>>
      forward_; // 双链表的next指针, 根据不同层级分层
  std::vector<std::weak_ptr<SkipListNode>>
      backward_; // 双链表的prev指针, 根据不同层级分层
  // ...
};
```

跳表结构（存一组由跳表结点组成的双链表）：

```c++

class SkipList {
private:
  std::shared_ptr<SkipListNode>
      head;              // 双链表额外的头结点
  int max_level;         // 跳表的最大层级数，限制跳表的高度
  int current_level;     // 跳表当前的实际层级数，动态变化
  size_t size_bytes = 0; // 跳表当前占用的内存大小（字节数），用于跟踪内存使用

  std::uniform_int_distribution<> dis_01;  // 随机层数的辅助生成器
  std::uniform_int_distribution<> dis_level;
  std::mt19937 gen;
};
```

### 2.2 CRUD

#### 增/改

这里我的策略是：

- 如果`put`的K，V不在跳表中，也就是要增加一个数据，那就要获得新结点要插入的层数，然后找到插入的位置，更新层之间的指针

- 如果`put`的K，V已经在跳表中了，那么就只更新结点数据，不更新层指针

##### 随机生成层数

用的是`std::mt19937 gen`和`std::uniform_int_distribution<> dis_01`两个变量

`gen`是一个随机数生成器，初始化：`gen = std::mt19937(std::random_device()())`

`dis_01`被初始化为均匀生成0，1的随机数字器，

`dis_01(gen)`类似于抛硬币，每次生成的数字为0,1，各有50%的概率

```c++
int SkipList::random_level() {
  // TODO: 实现随机生成level函数
  // 通过"抛硬币"的方式随机生成层数：
  // - 每次有50%的概率增加一层
  // - 确保层数分布为：第1层100%，第2层50%，第3层25%，以此类推
  // - 层数范围限制在[1, max_level]之间，避免浪费内存
  int level = 1;
  while (level < max_level && dis_01(gen) == 1) {
    level++;
  }
  return level;
}
```

每次新增数据都有可能生成更高的层数，通过随机数来控制每层的平均步长为`n / level_i`，n是结点数量，level_i是层级，从1开始

##### put

设插入K，V

- 从双链表头结点开始，从当前最高层开始遍历
- 记录每个层级第一个≤K的结点位置
- 找到最底层步长为1的双链表的第一个≤K的结点位置，如果该结点键与K相同，直接更新值为V，否则新插入结点自下往上，逐层更新层指针

```c++
void SkipList::put(const std::string &key, const std::string &value,
                   uint64_t tranc_id) {
  spdlog::trace("SkipList--put({}, {}, {})", key, value, tranc_id);

  // TODO: Lab1.1  任务：实现插入或更新键值对
  // ? Hint: 你需要保证不同`Level`的步长从底层到高层逐渐增加
  // ? 你可能需要使用到`random_level`函数以确定层数, 其注释中为你提供一种思路
  // ? tranc_id 为事务id, 现在你不需要关注它, 直接将其传递到 SkipListNode 的构造函数中即可
  // 老的value大小
  size_t old_size = 0;
  std::shared_ptr<SkipListNode> p = head;
  int cur_level = current_level - 1;
  std::vector<std::shared_ptr<SkipListNode>> insert_point(max_level, head);
  while (cur_level >= 0) {
    while (p->forward_[cur_level] != nullptr) {
      auto p_next = p->forward_[cur_level];
      if (p_next->key_ <= key) {
        p = p->forward_[cur_level];
      } else {
        break;
      }
    }
    insert_point[cur_level] = p;
    cur_level--;
  }
  // 如果有, 直接修改
  if (p != head && p->key_ == key) {
    old_size = p->value_.size();
    p->value_ = value;
  }
  // 否则的话, 直接插入
  else {
    int level = random_level();
    std::shared_ptr<SkipListNode> new_node = std::make_shared<SkipListNode>(key, value, level, tranc_id);
    size_bytes += key.size();
    current_level = std::max(current_level, level);
    // 找到了level0, 改变所有的insert_point的后继结点
    for (int level_i = 0; level_i < level; level_i++) {
      std::shared_ptr<SkipListNode> next = insert_point[level_i]->forward_[level_i];
      insert_point[level_i]->forward_[level_i] = new_node;
      new_node->forward_[level_i] = next;
      if(next != nullptr) {
        next->set_backward(level_i, new_node);
      }
      new_node->set_backward(level_i, insert_point[level_i]);
    }
  }
  size_bytes = size_bytes - old_size + value.size() + sizeof(uint64_t); 
}
```

#### 查



#### 删

### 2.3 迭代器

### 2.4 范围查询

### 思考题

1. 不同`Level`的链表的步长如何确定?
   - 最底层`Level 0`的链表步长肯定是1, 那么`Level 1`呢, `Level 2`呢?
2. 什么时候我们需要创建新的`Level`的?
   - 一开始的时候, `SkipList`的`Level`是多少?
   - 新创建的`Level`是逐层创建的吗? 还是说一次性提升好几层?
3. 上面演示的`SkipList`仅仅是单向链表, 如果是双向链表, 有什么区别呢?



