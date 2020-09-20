.. post:: Sep 20, 2020
   :author: JeffreyGuan
   :category: GoLang
   :location: BJ
   :tags: golang, waitgroup, error
.. :excerpt: 1

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-20-handle-errors-with-waitgroup.png
   :alt: 

巧妙使用WaitGroupt处理错误
=========================================
  
1. 写在前面
-----------

使用Go的众多好处之一是它在并发方面十分简单，而大家比熟悉的\ ``WaitGroups``\ 就是一个很好的例子。虽然在并发处理上十分的方便，但要想有效地处理并发和错误可能很棘手。

本篇文章旨在概述如何在不停止程序执行的情况下，运行多个goroutine并有效处理任何错误。

2. 具体实现
-----------

对于这个如何上，可以简单的概括为以下三点：

-  两个channel。这两个channel的作用是用于\ ``传递错误``\ 和\ ``传递WaitGroup何时完成``\ 。
-  一个groutine。主要作用是用于监听\ ``WaitGroup``\ 是否完成，如果完成了，将会关闭某个channel。
-  一个Select。它用于监听出现的错误或\ ``WaitGroup``\ 完成与否，无论谁先结束，那么Select就会先执行谁。

具体代码如下：

.. code-block:: go
   :caption: main.go
   :name: main.go
   :linenos:

    package main

    import (
        "errors"
        "fmt"
        "sync"
    )

    // ErrorHandler 返回一个错误。
    func ErrorHandler() error {
        return errors.New("generated errors")
    }

    func main() {
        // 创建两个channel，一个用于传递错误，另一个表示WaitGroup是否结束。
        errCh := make(chan error)
        wgCh := make(chan bool)

        var wg sync.WaitGroup

        wg.Add(2)

        go func() {
            fmt.Println("WaitGroup 1st.")

            // 这里可以定义我们需要执行的操作

            wg.Done()
        }()

        go func() {
            fmt.Println("WaitGroup 2nd")

            // 返回自定义的错误
            if err := ErrorHandler(); err != nil {
                errCh <- err
            }

            wg.Done()
        }()

        go func() {
            wg.Wait()
            close(wgCh)
        }()

        // 当有错误返回或WaitGroup执行结束时会被执行。
        select {
        case <-wgCh:
            break
        case err := <-errCh:
            close(errCh)
            panic(err)
        }

        fmt.Println("Main func ended!")
    }

运行上述代码，我们可以得到以下的输出，也从而能验证我们已经基于此达到了我们的目的。

.. code-block:: sh
   :caption: output
   :name: output
   :linenos:

    PS C:\Users\jeffrey\Desktop\hello> go run main.go
    WaitGroup 1st.
    WaitGroup 2nd
    panic: generated errors.

    goroutine 1 [running]:
    main.main()
            C:/Users/jeffrey/Desktop/hello/main.go:53 +0x2a6
    exit status 2
    PS C:\Users\jeffrey\Desktop\hello> 

--------------

欢迎关注我的微信公众号[double12gzh]：

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/wechat_public.jpg
   :alt: 