---
title: "聊聊JIT的指令重排序(Instruction Reorder)"           # 文章标题
author: "zhaob"              # 文章作者
description : "聊聊JIT的指令重排序(Instruction Reorder)"    # 文章描述信息
date: 2020-12-28            # 文章编写日期
lastmod: 2020-12-28         # 文章修改日期

tags : [                    # 文章所属标签
    "并发编程",
    "指令重排序",
    "JIT"
]
categories : [              # 文章所属标签
    "并发编程"
]
keywords : [                # 文章关键词
    "并发编程",
    "指令重排序"
]

next: /tutorials/github-pages-blog      # 下一篇博客地址
prev: /tutorials/automated-deployments  # 上一篇博客地址
---


# 聊聊JIT的指令重排序(Instruction Reorder)


在Java平台中，静态编译器（javac）是基本上不会执行指令重排序，但是JIT编译器则可能执行指令重排序，指令重排序会在运行时更改代码的执行顺序，可能会导致前后没有关联的语句无序的执行。
下面我们通过一个例子来展示JIT的指令重排序。

````Java
public class JitReorderDemo6 {
    public static void main(String[] args) {

        int round = 2000000;
        while (round-- > 0) {

            Stat stat = new Stat();
            //write thread
            new Thread(() -> {
                stat.a1 = 1;
                stat.b2 = 2;
                stat.c3 = 3;
                stat.d4 = 4;
                stat.e5 = 5;
                stat.f6 = 6;
            }).start();

            //read thread
            new Thread(() -> {
                int a = stat.a1;
                int b = stat.b2;
                int c = stat.c3;
                int d = stat.d4;
                int e = stat.e5;
                int f = stat.f6;

                if (a == 0 && b == 2) {
                    System.out.println("wtf a1=0,b2=2");
                }

                if (a == 0 && c == 3) {
                    System.out.println("wtf a1=0,c3=3");
                }

                if (b == 0 && c == 3) {
                    System.out.println("wtf b2=0,c3=3");
                }

                if (a == 0 && d == 4) {
                    System.out.println("wtf a1=0,d4=4");
                }

                if (a == 0 && e == 5) {
                    System.out.println("wtf a1=0,e5=5");
                }
                if (a == 0 && f == 6) {
                    System.out.println("wtf a1=0,f6=6");
                }

            }).start();
        }
        System.out.println("done");
    }


    static class Stat {
        int a1 = 0;
        int b2 = 0;
        int c3 = 0;
        int d4 = 0;
        int e5 = 0;
        int f6 = 0;
    }
}

````

示例的代码逻辑非常简单，`Stat`类有6个成员变量，初始化值都为0。`main`方法中有2个线程，分别为写线程和读线程。

写线程对应的操作为给stat对象的成员变量赋值，从a1-f6,分别*依次序*赋值为1-6。

读线程则读取stat对象的每一个成员变量，并检测部分异常情况的出现。

如果写线程代码是按照顺序执行的，那么成员变量的赋值总会是依照次序的，也就是a1=1必然发生在b2=2,c3=3,d4=4,e5=5,f6=6之前，读线程检测是否出现例如a1=0但是f6=6的情况，出现这种异常情况则说明f6=6这条语句在a=1之前执行了，说明JIT对指令进行了重排序。

由于JIT对字节码进行指令重排序并不是必然发生的，所以我们在`main`方法中进行多轮次的调用,调用2百万次，然后观察输出的情况。

````Java
//Output:
wtf a1=0,b2=2
wtf a1=0,c3=3
wtf a1=0,d4=4
wtf a1=0,e5=5
wtf a1=0,f6=6
wtf a1=0,b2=2
wtf a1=0,d4=4
wtf a1=0,e5=5
wtf a1=0,f6=6
done

````

通过输出我们可以观察到，b2-f6的赋值操作在实际运行过程中是有一定概率在赋值语句a1=1之前执行的,这种现象就是由于JIT进行指令重排序造成的。

为了更加清晰的查看JIT的指令重排序操作，我们可以使用hsdis来查看JIT生成的汇编代码。

通过观察源码，字节码和汇编代码，我们可以找到JIT的指令重排序的操作。


## 写线程中的重排序

写线程重排序示例如下：

![JIT指令重排序操作](http://qdb54dzqk.bkt.clouddn.com/聊聊JIT重排序-A.png)

我们可以观察到在写线程中，尽管源码和字节码的顺序是一致的，但是在JIT生成的汇编代码中,赋值语句的顺序发生了改变，这是由于JIT认为a1-f6语句没有相互关联，进行了重排序，重排序在单线程情况下不会有问题，但是在多线程情况下可能会引起线程安全问题，这种现象违反了线程安全性质中的*有序性*。


## 构造方法中的重排序


其实不仅写线程可能发生JIT指令重排序，通过观察汇编代码，我们可以发现在`Stat`类的构造方法中同样会发生指令重排，继续查看汇编代码如下：


![JIT指令重排序操作-Stat构造方法](http://qdb54dzqk.bkt.clouddn.com/%E8%81%8A%E8%81%8AJIT%E9%87%8D%E6%8E%92%E5%BA%8F-%E6%9E%84%E9%80%A0%E6%96%B9%E6%B3%95-B.png)

通过观察汇编代码我们可以得知在构造方法中，如果各数据没有关联，JIT有可能会对构造方法中的赋值语句进行重排序。

## 读线程中的重排序

接着观察读线程发生的JIT重排序，如下图：

![JIT指令重排序操作-Stat构造方法](http://qdb54dzqk.bkt.clouddn.com/%E8%81%8A%E8%81%8AJIT%E9%87%8D%E6%8E%92%E5%BA%8F-%E8%AF%BB%E7%BA%BF%E7%A8%8B-C.png)



## 小结

通过上面的示例，观察到JIT指令重排序对执行语句的影响，我们编写的程序源码和静态编译成成字节码顺序一般来说是一致的，但是这并不代表实际的执行顺序与源码是完全一致的，由于有JIT重排序的存在，执行顺序有可能与源码顺序不一致，从而导致多线程安全中的*有序性*问题（在没有正确的使用线程同步机制的情况下）。

重排序也并不是一定能够出现的，例如我们的demo运行了2百万轮次，才出现了10次左右的因为重排序导致的异常情况，但是这并不代表着程序是没有问题的，在并发环境下，JIT的重排序是为了提高程序的运行效率产生的，只要做到正确的使用线程同步机制就可以避免因为JIT指令重排造成的*有序性问题*。






