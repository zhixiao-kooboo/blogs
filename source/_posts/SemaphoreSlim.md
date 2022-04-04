---
title: 理解 SemaphoreSlim
date: 2022-04-04 19:16:35
categories: 
  - .Net基础
tags:
  - .Net
  - Multiple Threading
---
## 概述
Semaphore 的轻量级替代方案，它限制可以同时访问资源或资源池的线程数。

## 理解
在幼儿园里，因为体育室空间有限，老师设计了一个方案，使用 SemaphoreSlim 来控制有多少孩子可以在体育室玩耍。

他们在房间外面的地板上画了五对脚印。

当孩子们想要进体育室时，他们将鞋子留在一双空脚印上，然后进入房间。
一旦他们玩完他们就会出来，他们就会穿上自己的鞋子，并“释放”一个空脚印(Slot)。

<!-- more -->

如果一个孩子来了并且没有任何空脚印，他们就会去其他地方玩，或者只是呆一会儿，不时检查一下（没有 FIFO, LIFO 优先级）。

当老师在附近时，她会在走廊的另一边“释放”一排额外的 5 个脚印，这样 5 个以上的孩子就可以同时在房间里玩耍。
但是室外的空间有限（maximum count）, 老师能释放的脚印也是有限的。

它也有与 SemaphoreSlim 相同的“陷阱”
如果一个孩子玩完游戏并离开房间而没有穿上鞋子（不会触发“释放”），那么即使理论上有一个空脚印，脚印也会保持阻塞状态。
不过，孩子通常会被告知一定要穿上鞋子才能走。

有时一两个鬼鬼祟祟的孩子把他们的鞋子藏在别处并进入房间，即使所有的脚印都已经被占用了（即，SemaphoreSlim 并没有“真正”控制房间里有多少孩子）。
这通常不会很好地结束，因为房间的过度拥挤往往会导致孩子们哭泣和老师完全关闭房间。

## 例子
下面的例子创建了一个最大数量为三个线程和初始数量为零线程的semaphore。 我们启动了五个任务，所有这些任务都在等待semaphore。 主线程调用Release(Int32)重载将semaphore计数增加到最大值，允许三个任务进入semaphore。 每次释放semaphore时，都会显示之前的semaphore计数。 控制台消息跟踪semaphore的使用。 每个线程的模拟工作间隔略有增加，以使输出更易于阅读。
```C#
using System;
using System.Threading;
using System.Threading.Tasks;

public class Example
{
    private static SemaphoreSlim semaphore;
    // A padding interval to make the output more orderly.
    private static int padding;

    public static void Main()
    {
        // Create the semaphore.
        semaphore = new SemaphoreSlim(0, 3);
        Console.WriteLine("{0} tasks can enter the semaphore.",
                          semaphore.CurrentCount);
        Task[] tasks = new Task[5];

        // Create and start five numbered tasks.
        for (int i = 0; i <= 4; i++)
        {
            tasks[i] = Task.Run(() =>
            {
                // Each task begins by requesting the semaphore.
                Console.WriteLine("Task {0} begins and waits for the semaphore.",
                                  Task.CurrentId);
                
                int semaphoreCount;
                semaphore.Wait();
                try
                {
                    Interlocked.Add(ref padding, 100);

                    Console.WriteLine("Task {0} enters the semaphore.", Task.CurrentId);

                    // The task just sleeps for 1+ seconds.
                    Thread.Sleep(1000 + padding);
                }
                finally {
                    semaphoreCount = semaphore.Release();
                }
                Console.WriteLine("Task {0} releases the semaphore; previous count: {1}.",
                                  Task.CurrentId, semaphoreCount);
            });
        }

        // Wait for half a second, to allow all the tasks to start and block.
        Thread.Sleep(500);

        // Restore the semaphore count to its maximum value.
        Console.Write("Main thread calls Release(3) --> ");
        semaphore.Release(3);
        Console.WriteLine("{0} tasks can enter the semaphore.",
                          semaphore.CurrentCount);
        // Main thread waits for the tasks to complete.
        Task.WaitAll(tasks);

        Console.WriteLine("Main thread exits.");
    }
}
// The example displays output like the following:
//       0 tasks can enter the semaphore.
//       Task 1 begins and waits for the semaphore.
//       Task 5 begins and waits for the semaphore.
//       Task 2 begins and waits for the semaphore.
//       Task 4 begins and waits for the semaphore.
//       Task 3 begins and waits for the semaphore.
//       Main thread calls Release(3) --> 3 tasks can enter the semaphore.
//       Task 4 enters the semaphore.
//       Task 1 enters the semaphore.
//       Task 3 enters the semaphore.
//       Task 4 releases the semaphore; previous count: 0.
//       Task 2 enters the semaphore.
//       Task 1 releases the semaphore; previous count: 0.
//       Task 3 releases the semaphore; previous count: 0.
//       Task 5 enters the semaphore.
//       Task 2 releases the semaphore; previous count: 1.
//       Task 5 releases the semaphore; previous count: 2.
//       Main thread exits.
```

## 更多的理解
semaphore有两种类型：本地semaphore和named system semaphore。本地semaphore是应用程序的本地semaphore，系统semaphore在整个操作系统中都是可见的，适用于进程间同步。SemaphoreSlim 不使用Windows内核semaphore, 它是Semaphore类的轻量级替代方案。与Semaphore类不同，SemaphoreSlim 类不支持named system semaphore。只能将其用作本地semaphore。SemaphoreSlim 类是推荐用于在单个应用程序中同步的semaphore。

轻量级semaphore控制对应用程序本地资源池的访问。实例化semaphore时，可以指定可以同时进入semaphore的最大线程数。还可以指定可以同时进入semaphore的初始线程数。这定义了semaphore的计数。

每次线程进入semaphore时，计数递减，每次线程释放semaphore时，计数递增。要输入semaphore，线程调用 Wait 或 WaitAsync 重载之一。为了释放semaphore，它调用 Release 重载之一。当计数达到零时，对 Wait 方法之一的后续调用会阻塞，直到其他线程释放semaphore。如果多个线程被阻塞，则没有保证的顺序（例如 FIFO 或 LIFO）来控制线程何时进入semaphore。