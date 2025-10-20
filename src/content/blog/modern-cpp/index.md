---
title: 'modern C++ 个人总结'
publishDate: '2025-04-30'
updatedDate: '2025-10-20'
description: '记录现代c++(11/14/17/20)中的特性，使用一些简明的例子来展示'
tags:
  - modern c++
  - tutorial
language: '中文'
heroImage: { src: './cpp_icon.png', color: '#B4C6DA' }
---

# Modern C++

## reference（引用）

c++中的引用是一种给变量起别名的方法，类似于指针，原变量和引用将指向同一块内存地址

```c++
void add_three(int &a);
int a = 10;
int &b = a;	// b是a的别名
```

**指针引用**：顾名思义是对指针变量起别名，其形式通常为`type* &x`

```c++
// 可以这么理解，&b代表b是一个引用, 而int*是这个引用的类型
void change_pointer(int* &b);

int* a = new int;
int* &b = a;
```

## move_semantics（移动语义）

> 移动语义是指在c++对象中移动资源的这一动作，普通的对象拷贝可能花费不小的时间和空间资源，而**移动语义可以直接将对象A的资源转移给对象B，同时A的资源被清零**

由于移动针对**类对象（class object）**之间的过程，所以对普通类型（如int，double，裸指针）之间使用移动语义并不能移动资源，移动语义也不应该在普通类型中进行使用

移动语义离不开下面要介绍的左值和右值的概念，在c++中，任何一个**表达式**要么是左值要么是右值，例如：

```c++
// a 是左值, 10 是右值
int a = 10;
// b 是左值, a * 2 是右值
int b = a * 2;
```

### lvalue（左值）and rvalue（右值）

官方定义是，左值表示在内存中占据某个可识别位置的表达式（在表达式被计算完毕后仍然能够存在）

例子：

```c++
int add(int x, int y);
int a = 5;
int *p = &a;
int b = add(1, 3);
int &c = b;
vector &&d = {1,2,3}
```

在上面的代码中，`=`左边的表达式（变量）都是左值（`vector &&`表示一个`vector`的右值引用，后面会介绍）

与左值相反，是那些表达式被计算完毕后，无法继续存在的临时值，如字面量（字符串除外），或者一个函数的返回结果：

```c++
int add(int x, int y);
10;
add(1, 3);
2 * 4 + add(1, 3);
int x = 1, y = 2;
x * y;
```

上面的`10`，`add(1, 3)`，`x * y`都是右值

### 左值引用和右值引用

`reference`之前已经介绍过，就是给变量起别名

左值引用就是给左值变量起别名

右值引用就是给右值变量起别名，但右值在表达式结束后不应该存在，所以现代C++新的规则就是右值引用会延长引用的右值的生命周期，例如：

```c++
// 如果a是int类型，那么10在赋值表达式结束后，生命周期结束
int &&a = 10;
// 现在a是一个右值引用，并且引用了10，那么10在内存中不会消失，直到a消失后其再消失

// getString()会返回一个临时的string对象，如果s1是一个string类型，那么该对象在赋值结束后，生命周期结束
string getString();
string &&s1 = getString();
// 现在这个对象被s1所引用，同样的，该临时变量的生命周期只有在s1结束后才结束
```

**tips**：<u>一个右值引用的变量仍然是一个左值（因为在赋值表达式计算完毕后其仍能存在）</u>

### std::move

头文件`<utility>`中的`std::move`函数负责将一个左值无条件的转为右值，除此之外其什么都不做（不做资源的实际转移）

```c++
// 定义一个vector对象
std::vector<int> stealing_ints = {1, 2, 3}

// std::move()将stealing_ints变为一个右值，不产生新对象，也不改变原对象的内容
std::vector<int> &&rvalue_stealing_ints = std::move(stealing_ints);
// rvalue_stealing_ints是绑定在该右值的一个右值引用

// 因为是引用，所以可以同时使用stealing_ints和rvalue_stealing_ints来访问或修改该对象
std::cout << "Printing from stealing_ints: " << stealing_ints[1] << std::endl;
```

### 移动语义

移动语义通常指在对象之间利用移动资源的方式来代替复制资源的方式，比如下面这个例子：

```c++
// 定义一个vector
std::vector<int> int_array = {1, 2, 3, 4};

// 将int_array的资源移动到stealing_ints上 
std::vector<int> stealing_ints = std::move(int_array);
// 该表达式结束之后, int_array的资源就被清空了（如底层指针变为nullptr）
```

上面的代码首先将`int_array`转为一个右值，然后将其资源转移到`stealing_ints`中，在此表达时候不应该直接使用`int_array`（除非重新构建一个对象）

**为什么要先转为右值？**

资源的移动实际上是通过类的移动构造函数做到的，类似深拷贝函数，而移动构造函数要求参数为一个右值引用

下面是一个将对象移动到函数中的例子：

```c++
void move_add_three_and_print(std::vector<int> &&vec) {
  // 调用vector的移动构造函数，转移资源
  std::vector<int> vec1 = std::move(vec);
  vec1.push_back(3);
  for (const int &item : vec1) {
    std::cout << item << " ";
  }
  std::cout << "\n";
}
int main() {
	std::vector<int> int_array2 = {1, 2, 3, 4};
    // vec右值引用，引用了int_array2
    move_add_three_and_print(std::move(int_array2));
    // 之后不应该直接使用int_array2,因为其资源已经被移动
}
```

**为什么要在函数中再次调用`std::move`？**

`vec`是一个右值引用，但其本身是一个左值，而移动构造函数接受一个右值作为参数，所以需要将`vec`先转为右值再赋值给`vec1`才能够正确转移，否则会调用`vector`的拷贝构造函数

具体可以参考下面这段代码（来源：[现代c++教程](https://changkun.de/modern-cpp/zh-cn/03-runtime/#%E5%8F%B3%E5%80%BC%E5%BC%95%E7%94%A8%E5%92%8C%E5%B7%A6%E5%80%BC%E5%BC%95%E7%94%A8)）：

```c++
#include <iostream>
#include <string>

void reference(std::string& str) {
    std::cout << "左值" << std::endl;
}
void reference(std::string&& str) {
    std::cout << "右值" << std::endl;
}

int main()
{
    std::string lv1 = "string,"; // lv1 是一个左值
    // std::string&& r1 = lv1; // 非法, 右值引用不能引用左值
    std::string&& rv1 = std::move(lv1); // 合法, std::move可以将左值转移为右值
    std::cout << rv1 << std::endl; // string,

    const std::string& lv2 = lv1 + lv1; // 合法, 常量左值引用能够延长临时变量的生命周期
    // lv2 += "Test"; // 非法, 常量引用无法被修改
    std::cout << lv2 << std::endl; // string,string,

    std::string&& rv2 = lv1 + lv2; // 合法, 右值引用延长临时对象生命周期
    rv2 += "Test"; // 合法, 非常量引用能够修改临时变量
    std::cout << rv2 << std::endl; // string,string,string,Test

    reference(rv2); // 输出左值

    return 0;
}
```

`reference(rv2)`实际输出的是左值，也就代表着其实际调用了`void reference(std::string&amp; str)`函数

## move_constructors（移动构造）

移动构造函数和移动赋值运算符是在类内部实现的方法，用于将一个对象的资源转移到另一个对象

```c++
class Person {
public:
  Person() : age_(0), nicknames_({}), valid_(true) {}

  // 构造函数通过接受一个std::vector<std::string>右值作为参数
  // 这使得构造时比一般的会更快，因为不需要复制vector的底层数组
  Person(uint32_t age, std::vector<std::string> &&nicknames)
      : age_(age), nicknames_(std::move(nicknames)), valid_(true) {}

  // Person类的移动构造函数：接受一个Person类型的右值
  // 具体过程是：将传入的右值中的对象成员，通过std::move()转为右值
  // 再调用vector的移动构造函数，实现资源转移
  
  // vector是如何转移资源的？
  // 假设vector有一个void* arr指针用于指向开辟的动态数组
  // nicknames_(std::move(person.nicknames_))实际上做：
  // 1. nicknames_.arr = person.nicknames_.arr;
  // 2. person.nicknames_arr = nullptr;
  Person(Person &&person)
      : age_(person.age_), nicknames_(std::move(person.nicknames_)),
        valid_(true) {
    std::cout << "Calling the move constructor for class Person.\n";
    // 将被移动对象的实例标志置为false
    person.valid_ = false;
  }

  // Person类的移动赋值运算符，原理与移动构造函数相同
  Person &operator=(Person &&other) {
    std::cout << "Calling the move assignment operator for class Person.\n";
    age_ = other.age_;
    nicknames_ = std::move(other.nicknames_);
    valid_ = true;

    // 将被移动对象的实例标志置为false
    other.valid_ = false;
    return *this;
  }

  // delete关键字代表将类的指定构造函数删除
  // Person类将不再有拷贝构造函数
  Person(const Person &) = delete;
  Person &operator=(const Person &) = delete;

  uint32_t GetAge() { return age_; }

  // 返回类型中的这个 & 符号意味着我们返回对 nicknames_[i] 处字符串的引用。
  // 这也意味着我们不拷贝结果字符串，
  // 此函数返回的内存地址实际上是指向向量 nicknames_ 内存的地址。
  std::string &GetNicknameAtI(size_t i) { return nicknames_[i]; }

  void PrintValid() {
    if (valid_) {
      std::cout << "Object is valid." << std::endl;
    } else {
      std::cout << "Object is invalid." << std::endl;
    }
  }

private:
  uint32_t age_;
  std::vector<std::string> nicknames_;
  // 跟踪对象的数据是否有效，即是否所有数据都已移动到另一个实例。
  bool valid_;
};
```

测试：

```c++
int main() {
  // 创建一个Person对象，此时调用的是Person(uint32_t age, std::vector<std::string> &&nicknames)构造函数
  Person andy(15445, {"andy", "pavlo"});
  std::cout << "Printing andy's validity: ";
  andy.PrintValid();

  // 将andy对象转为右值，并使用移动赋值运算符将andy的资源移动到andy1
  Person andy1;
  andy1 = std::move(andy);

  // 此时andy1的资源是有效的，而andy的资源已经被转移了，不再有效
  std::cout << "Printing andy1's validity: ";
  andy1.PrintValid();
  std::cout << "Printing andy's validity: ";
  andy.PrintValid();

  // 该表达式使用Person类的移动构造函数
  
  Person andy2(std::move(andy1));

  // 跟上面一样，该表达式结束后，andy2将拥有andy1的资源，andy1资源被转移，不再有效
  // 后续不应该继续使用andy1，除非重新初始化andy1，
  std::cout << "Printing andy2's validity: ";
  andy2.PrintValid();
  std::cout << "Printing andy1's validity: ";
  andy1.PrintValid();

  // 拷贝构造函数和拷贝运算符函数被删除，下面的代码都会报编译错误
  // Person andy3;
  // andy3 = andy2;

  // Person andy4(andy2);

  return 0;
}
```

## C++ Templates（C++模板）

## 参考资料
- [C++之旅](https://book.douban.com/subject/36596125/)
- [CMU-bootcamp 翻译版](https://book.douban.com/subject/36596125/)