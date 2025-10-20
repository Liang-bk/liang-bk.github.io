---
title: 'c++速转java'
publishDate: '2025-10-20'
updatedDate: '2025-10-20'
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
         System.out.println("方法调用后 x 的值: " + x); // 输出: 10
     }
     public static void modifyValue(int value) {
         value = 20; // 修改的是副本，不影响原始变量
     }
  }
  ```

- java class类型如`String`，类似c++指针参数（但传递的时候显示的还是对象），传入的是指向对象的指针，可以在方法内修改对象，但对该对象整体重新赋值则相当于c++中让函数内指针指向新内存地址，不会改变外部对象

  ```c++
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

- java引用传递，即传数组，与c++行为一致

