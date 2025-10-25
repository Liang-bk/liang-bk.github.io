---
title: 'c++速转java'
publishDate: '2025-10-20'
updatedDate: '2025-10-25'
description: '从现代c++语法快速过渡java语法'
tags:
  - modern c++
  - java
  - tutorial
language: '中文'
---

# c++速转java

## 传参

- java普通类型如`int`，`double`，`char`，与c++相同，都是值传递，方法内修改不影响外部传入的值

  ```java
  public class ValuePassing {
     public static void main(String[] args) {
         int x = 10;
         modifyValue(x);
         // x仍然为10
     }
     public static void modifyValue(int value) {
         value = 20; // 修改的是副本，不影响原始变量
     }
  }
  ```

  ```c++
  void func() {
      int x = 10;
      modifyValue(x);
      // x仍然为10
  }
  void modifyValue(int value) {
      value = 20;
  }
  ```

- java class类型如`String`，类似c++指针参数（但传递的时候显示的还是对象），传入的是指向对象的指针，可以在方法内修改对象，但对该对象整体重新赋值则相当于c++中让函数内指针指向新内存地址，不会改变外部对象

  ```java
  public class Switch {
      public static void main(String[] args) {
          String x1 = "10";
          String y1 = "20";
          System.out.println("交换前x1 = "+x1);
          System.out.println("交换前y1 = "+y1);
  		// 进行数据交换
          swap(x1, y1);
  		// x1, y1不会变化
          System.out.println("交换后x1 = "+x1);
          System.out.println("交换后y1 = "+y1);
   
   
      }
   	// 在c++中相当于swap(string* x, string* y)
      public static void swap(String x, String y) {
          // 跟c++类似，传递的指针实际是复制了一份指针指向对象
          // java中重新赋值相当于c++让复制的那份指针指向别的内存，但原指针不变
          String z ;
          z = x;
          x = y;
          y = z;
          System.out.println("x = "+x);
          System.out.println("y = "+y);
      }
  }
  ```

  ```c++
  void func() {
      string* s1 = new string("123");
      string* s2 = new string("456");
      swap(s1, s2);
      // s1, s2的指向仍然没变
  }
  void swap(string* s1, string* s2) {
      string* temp = s1;
      s1 = s2;
      s2 = temp;
  }
  ```

- java引用传递，即传数组，与c++行为一致

## 继承

## 多态

### 抽象类

### 接口

### lambda表达式

二者形式基本相同，但本质不太一样，c++的lambda表达式本质是重载了`()`的对象，也可以叫做函数对象，java则是对匿名内部类的一种简化写法

1. java lambda：本质是实现接口的方法

   实现条件：

   - 接口
   - 单一方法

   或者用`@FunctionalInterface`来注解一个接口

   ```java
   @FuntionalInterface
   interface Swim {
       void swimming();
   }
   
   public static void main(String[] args) {
       Swim swim = () -> {
           System.out.println("swim...");
       };
   }
   ```

   如上所示，当某个方法使用`Swim`接口做参数时，可以这么写：

   ```java
   void func1(Swim s) {
       ...
   }
   void call_func1() {
       func1(() -> {
           System.out.println("swim...");
       });
   }
   ```

   由于lambda表达式是对应接口的实现，所以`()`是接口方法的形参列表，函数体返回值也要和接口方法保持一致

   某些情况下有可简化的lambda表达式（比如方法引用这种跟c++的作用域长的差不多但不怎么相关的玩意），但我个人觉得没什么必要，一般情况下使用上述lambda形式就够了，非简化不可的时候idea可以智能简化

2. c++ lambda：

   以`std::sort()`为例：

   ```c++
   std::vector<int> v {1, 2, 5, 4, 3};
   std::sort(v.begin(), v.end(), [](int x, int y) -> bool {
       return x > y;
   });
   ```

   形式为：

   ```c++
   // -> return_type可省略
   auto lambda = [捕获](形参列表) -> return_type {
       函数体
   };
   ```

   本质是把对象当函数用，没有java的必须实现接口的限制，还多了可捕获列表，不过二者在写法上大差不差

## 泛型



## 常用数据结构和API



## IO流



## 线程





