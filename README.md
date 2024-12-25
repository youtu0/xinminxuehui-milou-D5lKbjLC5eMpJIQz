
## 一：背景


### 1\. 讲故事


上一篇我们用 `Thread.Sleep` 的方式演示了线程池饥饿场景下的动态线程注入，可以观察到大概 1s 产生 `1~2` 个新线程，很显然这样的增长速度扛不住上游请求对线程池的DDOS攻击，导致线程池队列越来越大，但C\#团队这么优秀，能优化的地方绝对会给大家尽可能的优化，比如这篇我们聊到的 `Task.Result` 场景下的注入。


## 二：Task.Result 角度下的动态注入


### 1\. 测试代码


为了直观的体会到优化效果，先上一段测试代码观察一下。



```

        static void Main(string[] args)
        {
            for (int i = 0; i < 10000; i++)
            {
                ThreadPool.QueueUserWorkItem((idx) =>
                {
                    Console.WriteLine($"{DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss:fff")} -> {idx}: 这是耗时任务");

                    try
                    {
                        var client = new HttpClient();
                        var content = client.GetStringAsync("https://youtube.com").Result;
                        Console.WriteLine(content.Length);
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine(ex.Message);
                    }
                }, i);
            }

            Console.ReadLine();
        }


```

![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241224133104298-343804747.png)


从卦象上来看大概1s产生4个新线程，再仔细看的话大概是250ms一个，虽然250不大好听，但不管怎么说确实比 `Thread.Sleep` 场景下只产生 `1~2` 个线程要快了好几倍，以终为始，我们再反向的看下这个优化的底层逻辑在哪？


### 2\. 底层逻辑在哪里


还是那句话，千言万语不抵一张图，流程图大概如下：


![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241224133104328-1182807567.png)


接下来解释下其中的几个元素。


1. NotifyThreadBlocked


这是主动通知 GateThread 线程赶紧醒来，通过上一篇的知识大家应该知道 GateThread 会500ms一次被动唤醒，但为了提速不可能再这么干了，需要让人强制唤醒它，修剪后的源码如下：



```

    private bool SpinThenBlockingWait(int millisecondsTimeout, CancellationToken cancellationToken)
    {
        var mres = new SetOnInvokeMres();

        AddCompletionAction(mres, addBeforeOthers: true);

        bool notifyWhenUnblocked = ThreadPool.NotifyThreadBlocked();

        var returnValue = mres.Wait((int)(millisecondsTimeout - elapsedTimeTicks), cancellationToken);

        return returnValue;
    }

    public bool NotifyThreadBlocked()
    {
        GateThread.Wake(this);

        return true;
    }

    public static void Wake(PortableThreadPool threadPoolInstance)
    {
        DelayEvent.Set();
    }


```

卦中的 `DelayEvent.Set();` 正是强制唤醒 GateThread 的 event 事件。


2. HasBlockingAdjustmentDelayElapsed


GateThread 是注入线程的官方通道，那到底要不要注入线程呢？肯定少不了一些判断，其中一个判断就是当前的延迟周期是否超过了 250ms，这个250ms的阈值最终由 `BlockingConfig.MaxDelayMs` 变量指定，这是能否调用 `CreateWorkerThread`方法需要闯的一个关口，参考代码如下：



```

        private static class BlockingConfig
        {
            MaxDelayMs =(uint) AppContextConfigHelper.GetInt32Config(
                        "System.Threading.ThreadPool.Blocking.MaxDelayMs",
                            250,
                            false);
        }

        private static void GateThreadStart()
        {
            while (true)
            {
                bool wasSignaledToWake = DelayEvent.WaitOne((int)delayHelper.GetNextDelay(currentTimeMs));
                currentTimeMs = Environment.TickCount;

                do
                {
                    previousDelayElapsed = delayHelper.HasBlockingAdjustmentDelayElapsed(currentTimeMs, wasSignaledToWake);
                    if (pendingBlockingAdjustment == PendingBlockingAdjustment.WithDelayIfNecessary && !previousDelayElapsed)
                    {
                        break;
                    }

                    uint nextDelayMs = threadPoolInstance.PerformBlockingAdjustment(previousDelayElapsed);

                } while (false);
            }
        }

        public bool HasBlockingAdjustmentDelayElapsed(int currentTimeMs, bool wasSignaledToWake)
        {
            if (!wasSignaledToWake && _adjustForBlockingAfterNextDelay)
            {
                return true;
            }

            uint elapsedMsSincePreviousBlockingAdjustmentDelay = (uint)(currentTimeMs - _previousBlockingAdjustmentDelayStartTimeMs);
            return elapsedMsSincePreviousBlockingAdjustmentDelay >= _previousBlockingAdjustmentDelayMs;
        }


```

从上面的代码可以看到一旦 `previousDelayElapsed =false` 就直接 break 了，不再调用`PerformBlockingAdjustment` 方法来闯第二个关口。


3. PerformBlockingAdjustment


一旦满足了250ms阈值之后，接下来就需要观察ThreadPool当前的负载能力，由内部的 `ThreadCounts` 提供支持，比如 NumProcessingWork 表示当前线程池正在处理的任务数， NumThreadsGoal 表示线程不要超过此上限值，如果超过了就进入动态注入阶段，参考代码如下：



```

    private struct ThreadCounts
    {
        public short NumProcessingWork;
        public short NumExistingThreads;
        public short NumThreadsGoal;
    }


```

有了这个基础之后，接下来再上一段注入线程需要满足的第二个关口。



```

        private static void GateThreadStart()
        {
            uint nextDelayMs = threadPoolInstance.PerformBlockingAdjustment(previousDelayElapsed);
        }

        private uint PerformBlockingAdjustment(bool previousDelayElapsed)
        {
            var nextDelayMs = PerformBlockingAdjustment(previousDelayElapsed, out addWorker);

            if (addWorker)
            {
                WorkerThread.MaybeAddWorkingWorker(this);
            }
            return nextDelayMs;
        }

        private uint PerformBlockingAdjustment(bool previousDelayElapsed, out bool addWorker)
        {
            if (counts.NumProcessingWork >= numThreadsGoal && _separated.numRequestedWorkers > 0)
            {
                addWorker = true;
            }
        }


```

从卦中代码可以看到，一旦线程池中 `处理的任务数 >= 线程上限值`，这就表示当前线程池正在满负荷的跑，`numRequestedWorkers>0` 表示有新任务来了需要线程来处理，所以这两组条件一旦满足，就必须要创建新线程。


### 3\. 如何眼见为实


刚才啰嗦了那么多，那如何眼见为实呢？非常简单，还是用 dnspy 的断点日志功能观察，我们下三个断点。


1. 第一个条件 HasBlockingAdjustmentDelayElapsed 处增加 `1. {!wasSignaledToWake} {this._adjustForBlockingAfterNextDelay}, 延迟时间：{currentTimeMs - this._previousBlockingAdjustmentDelayStartTimeMs} ，上一次延迟：{_previousBlockingAdjustmentDelayMs}`。


![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241224133104272-2011797635.png)


2. 第二个条件 PerformBlockingAdjustment 处增加 `2. 正在处理任务数：{threadCounts.NumProcessingWork} ，合适线程数：{num}，是否要新增线程：{this._separated.numRequestedWorkers>0}` 。


![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241224133104310-2124517114.png)


3. 线程创建 WorkerThread.CreateWorkerThread 处增加 `3. 已成功创建线程`  。


![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241224133104309-570134937.png)


最后把程序跑起来，观察 output窗口 的结果，非常清爽，吉卦。


![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241224133104293-281949744.png)


## 三：总结


采用主动通知的方式唤醒GateThread可以让每秒线程注入数由原来的 `1~2` 个提升到 `4` 个，虽然有所优化，但面对上游洪水猛兽般的请求，很显然也是杯水车薪，最终还是酿成了线程饥饿的悲剧，下一篇我们继续研究如何让线程注入的快一点，再快一点。。。
![图片名称](https://images.cnblogs.com/cnblogs_com/huangxincheng/345039/o_210929020104%E6%9C%80%E6%96%B0%E6%B6%88%E6%81%AF%E4%BC%98%E6%83%A0%E4%BF%83%E9%94%80%E5%85%AC%E4%BC%97%E5%8F%B7%E5%85%B3%E6%B3%A8%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)


 本博客参考[楚门加速器官网](https://chuanggeye.com)。转载请注明出处！
