.. post:: Sep 21, 2020
   :author: 大海星
   :category: GoLang
   :location: BJ
   :tags: golang, waitgroup, error
.. :excerpt: 1

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-21-go-switch/0.png
   :alt:

goroutine切换背后那些事儿
============================

本文基于于GoLang 1.13。

1. 写在前面
-----------

    微信公众号：\ **[double12gzh]**

    个人主页: https://gzh.readthedocs.io

    关注容器技术、关注\ ``Kubernetes``\ 。问题或建议，请公众号留言。

Goroutine很轻量，从资源消耗方面来看，它只需要一个2Kb的内存栈就可以运行；从运行时来看，它的运行成本也很低，将一个goroutine切换到另一个goroutine并不需要很多操作。

在进行讲解golang的切换之前，我们先High
Level的看一下goroutine切换的相关内容。

2. 案例
-------

golang会根据两种断点将goroutine调度到线程上：

-  当一个goroutine阻塞了。如：系统调用，mutex，或者通道。被阻塞的goroutine会进入睡眠模式/队列，让Go调度并运行一个等待的goroutine。被阻塞的goroutine进入睡眠模式/队列，允许Go调度 
和运行一个等待的goroutine。

-  在函数调用过程中，假如goroutine必须增长它的栈。这个断点允许Go调度另一个goroutine，避免正在运行的那个goroutine占用CPU。

在这两种情况下，运行调度器的g0会用另一个准备运行的goroutine替换当前的goroutine。然后，被选中的goroutine取代g0，从而在线程上运行。

    如果您想了解更多关于\ ``g0``\ 的内容，请参考\ `g0 <https://www.cnblogs.com/double12gzh/p/13661777.html>`__

将一个运行中的goroutine切换到另一个运行中的goroutine涉及到两个切换。

-  ``g``-> \ ``g0``\ 。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-21-go-switch/1.png
   :alt:

-  ``g0``-> 另一个\ ``g``\ 。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-21-go-switch/2.png
   :alt:

在GoLang中，groutine真的非常轻量。为了保存，它只需要两个东西：

-  goroutine是在哪一行停止的。即：在被调度前，goroutine是在哪一行停止的，当前要运行的指令被记录在程序计数器（PC）中。goroutine稍后将在同一点恢复。

-  存放goroutine的堆栈。这个堆栈的目的是为了方便再次运行时恢复其局部变量。

下面我们深入看一下。

3. PC（程序记数器）
-------------------

为了便于举例，我将使用一个通过channel进行通信的goroutine来说明，这两个goroutine中，一个可以产生数据的，其它的用于消费数据。代码如下：

.. code-block:: go
   :caption: main.go
   :name: main.go
   :linenos:

    package main

    import (
        "fmt"
        "sync"
    )

    const COUNT = 100

    func main() {
        var wg sync.WaitGroup

        c := make(chan int, 10)

        wg.Add(1)

        // 生产数据
        go func() {
            for i := 0; i < COUNT; i++ {
                c <- i
            }

            close(c)
            wg.Done()
        }()

        // 消费数据
        for i := 0; i < 3; i++ {
            wg.Add(1)

            go func() {
                for v := range c {
                    if v%2 == 0 {
                        fmt.Println(v)
                    }
                }
            }()
        }

        wg.Wait()
    }

消费者基本上会打印0到99的偶数，我们将重点关注第一个goroutine--生产者--向缓冲区添加数字。当缓冲区满了，它将在发送消息时阻塞。此时，Go要切换到g0，调度另一个goroutine。

如前所述，Go首先需要保存当前指令，以便在同一指令处恢复goroutine。程序计数器(PC)保存在goroutine的内部结构中。

上面的代码可以使用以下图来简单说明：

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-21-go-switch/3.png
   :alt:

指令和它们的地址可以通过命令获取：

.. code:: bash

    ➜  hello go tool compile -N -l main.go
    ➜  hello ls | grep main.o
    main.o

下面是生产者的一个示例：

.. code:: bash

    ➜  hello go tool objdump main.o

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-21-go-switch/4.png
   :alt:

在函数\ ``runtime.chansend1``\ 上阻塞通道前，程序逐条指令执行。Go将当前的程序计数器保存到当前goroutine的内部属性中。在我们的例子中，Go保存程序计数器的地址是\ ``0x4268d0``\ ，这 
个地址是在\ ``runtime``\ 和方法\ ``runtime.chansend1``\ 内部的。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-21-go-switch/5.png
   :alt:

然后，当\ ``g0``\ 唤醒goroutine时，它将在同一指令处恢复，对数值进行循环并推入通道。

下面我们来谈谈goroutine切换过程中的栈管理。

4. 栈(stack)
------------

在被阻塞之前，正在运行的goroutine有它的原始栈。这个堆栈包含临时内存，比如变量i:

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-21-go-switch/6.png
   :alt:

然后，当它在通道上阻塞时，goroutine将和它的堆栈一起切换到g0，这个goroutine将会有一个更大的栈。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-21-go-switch/7.png
   :alt:

在切换之前，堆栈将被保存，以便在goroutine再次运行时恢复。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-21-go-switch/8.png
   :alt:

我们现在已经完整地了解了goroutine切换中涉及的不同操作。现在让我们看看它是如何影响性能的。

我们应该注意到，一些架构(比如arm)需要多保存一个寄存器LR(链接寄存器)。

5. 操作
-------

为了测量goroutine切换可能需要的时间，我们将使用前面写的程序。然而，它并不能给出一个完美的性能视图，因为它可能取决于找到下一个要调度的goroutine所需的时间。这样goroutine的切换也会
影响性能，从函数prolog的切换比从通道上阻塞的goroutine切换要做的操作更多。

我们来总结一下我们要测量的操作：

-  当前的g在通道上阻塞并切换到g0:

   -  PC和堆栈指针一起被保存在一个内部结构中
   -  g0被设置为正在运行的goroutine。
   -  g0的堆栈取代了当前的堆栈。

-  g0正在寻找一个新的goroutine来运行。
-  g0必须与所选的goroutine进行切换。

   -  PC和堆栈指针被从内部结构中提取出来。
   -  程序跳转到获取的PC地址。

如下图：

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-21-go-switch/9.png
   :alt:

从\ ``g``\ 到\ ``g0``\ 或\ ``g0到``\ g的切换是最快的阶段。它们包含少量固定的指令，这一点与调度器检查许多源以寻找下一个要运行的goroutine的情况相反。根据运行程序的情况，这个阶段甚
至可能需要更多的时间。

需要说明的一点是，对于以上测试的结果会因机器架构的不同而不同

--------------

欢迎关注我的微信公众号：

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/wechat_public.jpg
   :alt: