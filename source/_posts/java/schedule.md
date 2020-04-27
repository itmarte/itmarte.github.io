---
title: schedule
tags:
  - java
  - schedule
categories:
  - java
  - schedule
date: 2020-04-27 13:47:12
---

并发编程领域中**定时器** 相关内容经常被一些介绍并发编程书籍所遗忘，属于并发编程学习优先级较低的知识点。在JDK源码中有两种定时器实现，一种是JDK1.3引入的**\*Timer**类*，它是一种基于单线程操作的简单任务调度器，虽然存在较多设计缺陷，但仍有很多应用场景和使用案例；另一种JDK1.5引入的**ScheduledThreadPoolExecutor**类，是一种基于线程池操作的较复杂任务调度器，同时也是官方推荐的任务调度器实现。

定时器Timer，也称简单任务调度器。它由以下四个类组成，

- 定时任务（TimerTask类）
- 任务队列（TaskQueue类）
- 定时线程（TimerThread类）
- 定时器（Timer类）

## **定时任务**

```
public abstract class TimerTask implements Runnable {
    final Object lock = new Object();

    //任务状态
    int state = VIRGIN;
    static final int VIRGIN = 0;
    static final int SCHEDULED   = 1;
    static final int EXECUTED    = 2;
    static final int CANCELLED   = 3;
    
    //下次执行时间
    long nextExecutionTime;
    //调度至执行间隔时间
    long period = 0;
}
```

抽象类TimerTask实现Runnable接口，表明该类作为定时任务模版，用户可以根据业务场景定义具体任务。TimerTask类要维护**任务状态** (state)、**任务下次执行时间**（nextExecutionTime）和**任务调度至执行的间隔时间**（period）。

> 任务状态

定时器任务生命周期中可能处于下表所示的4种不同的状态，在给定的时刻定时器任务只能处于其中一种状态。

![img](https://pic1.zhimg.com/80/v2-156e3cab41f11ad251640b4286e33cd4_720w.jpg)

> 执行任务

TimerTask类的抽象方法run来自Runnable接口，TimerTask并未实现该接口，延迟至子类实现。用户可在派生类中自定义任务逻辑。

```
public abstract void run();
```

抽象类TimerTask的run方法并不一定要来源于Runnable接口，它并未接受线程调度，而是由TimerThread线程从TimerQueue中消费任务，然后直接调用TimerTask.run()执行任务。基于这种理解，TimerTask类完全可以像这样定义：

```
public abstract class TimerTask {  // 舍去implement Runnable

    //由抽象类自己定义,而非来自Runnable接口
    public abstract void run();
}
```

TimerTask类这种写法可以理解为被**过度设计**了，读者可思之。

> 取消任务

如果当前任务正处于SCHEDULED状态，允许撤销当前任务，置任务为CANCELLED状态，返回true表示任务撤销成功；若任务处于其它状态，也置任务为CANCELLED状态，并返回false表示任务撤销失败。

```
public boolean cancel() {
    synchronized(lock) {
        boolean result = (state == SCHEDULED);
        //实际上所有任务都能被取消
        state = CANCELLED;
        return result;
    }
}
```

调用TimerTask.cancel()，虽然对不同状态有不同的返回值，但不管什么状态都能够被取消。设计逻辑匪夷所思，我认为这种**设计不合理**，读者可思之。

> 调度执行时间

scheduledExecutionTime方法获取任务被调度后最近的开始执行时间点，保证调度时间在下次执行时间之前。

```
public long scheduledExecutionTime() {
    synchronized(lock) {
        return (period < 0 ? 
            nextExecutionTime + period : nextExecutionTime - period);
    }
}
```

## **定时线程**

从优先级队列里异步消费任务的操作由单线程完成。TimerThread是单线程，因此需要mainLoop循环逻辑来轮询消费任务队列。

```
class TimerThread extends Thread {

    boolean newTasksMayBeScheduled = true;

    //内部维护一个队列
    private TaskQueue queue;

    TimerThread(TaskQueue queue) {
        this.queue = queue;
    }
}
```

> 轮询任务

```
@Override
public void run() {
    try {
        //循环执行逻辑
        mainLoop();
    } finally {
        synchronized(queue) {
            newTasksMayBeScheduled = false;
            //清空任务队列. 在结束循环后可能仍有任务被加入到队列,因此需要清空.
            queue.clear();
        }
    }
}

private void mainLoop() {
    while (true) {
        try {
            TimerTask task;
            boolean taskFired;
            synchronized(queue) {  
                //若队列为空且定时器未被撤销,则挂起定时线程直至被唤醒       
                while (queue.isEmpty() && newTasksMayBeScheduled) {                     
                    queue.wait();
                }
                //若线程被唤醒后队列仍为空,则结束循环. 说明此时定时器被撤销.
                if (queue.isEmpty()) {
                    break;            
                }    
  
                long currentTime, executionTime;
                //获取最近执行时间任务
                task = queue.getMin();
                synchronized(task.lock) {
                    //任务若被取消,则从队列中移除,并继续轮询
                    if (task.state == TimerTask.CANCELLED) {
                        queue.removeMin();
                        continue;
                    }

                    currentTime = System.currentTimeMillis();
                    executionTime = task.nextExecutionTime;
                    //任务最近要执行
                    if (taskFired = (executionTime<=currentTime)) {
                        //若为非重复执行任务,从队列中移除该任务,并设置该任务状态为已执行
                        if (task.period == 0) {
                            queue.removeMin();
                            task.state = TimerTask.EXECUTED;
                        } else {
                            //若为重复执行任务,则在指定时刻重新调度该任务
                            queue.rescheduleMin(
                                task.period<0 ? currentTime-task.period
                                    : executionTime + task.period);                      
                        }
                    }
                    //若最近无任务要执行,则等待至要执行任务的指定时刻
                    if (!taskFired) {
                        queue.wait(executionTime - currentTime);
                    }
                }
            }
                
            //任务已释放,运行任务
            if (taskFired) { 
                task.run();
            }
        } catch(InterruptedException e) {
        }
    }
}
```

## **任务队列**

任务队列是基于完全二叉树实现的小顶堆。队列初始容量为128，由于0位置不存储任务，因此实际初始容量为127，size表示队列的任务数。

```
class TaskQueue {

    //基于顺序表实现的定时任务队列
    private TimerTask[] queue = new TimerTask[128];

    //队列任务数
    private int size = 0;
}
```

> 查询容量

查询队列任务数和判断队列是否为空都直接使用任务队列内部维护的size属性，因此这两个操作的时间复杂度为O(1)。

```
/** 队列任务数 */
int size() { return size; }

/** 队列是否为空 */
boolean isEmpty() { return size==0; }
```

> 添加任务

主线程向任务队列中注入新任务。如果当前任务队列容量已达极限，则在原容量基础上扩容一倍，并在任务队列末尾追加新任务，并根据任务执行时间作为优先级调整新任务在任务队列中的位置。

```
/** 新增任务并调整小顶堆 */
void add(TimerTask task) {
    //任务数达到队列最大容量,则扩容一倍
    if (size + 1 == queue.length) {
        queue = Arrays.copyOf(queue, 2*queue.length);
    }
    //添加任务
    queue[++size] = task;
    //向上调整任务
    fixUp(size);
}
```

![img](https://pic3.zhimg.com/80/v2-3e64d663f2d599d0c7b40a7464dd0072_720w.jpg)

> 获取任务

从任务队列中获取最近将要执行任务的时间复杂度为O(1)；获得指定位置任务的时间复杂度也是O(1)。

```
/** 获得下次执行时间最小的任务,即最小堆根结点 */
TimerTask getMin() { return queue[1]; }

/** 获得指定位置的任务 */
TimerTask get(int i) { return queue[i]; }
```

![img](https://pic4.zhimg.com/80/v2-1a80040c19ea94d2d18ef6452ae64183_720w.jpg)

> 移除任务

```
/** 移除下次执行时间最小的任务,即移除堆顶任务 */
void removeMin() {
    queue[1] = queue[size];
    queue[size--] = null;
    fixDown(1);
}
```

![img](https://pic4.zhimg.com/80/v2-d0a23892cf5a8aaef780bf8f0e6f0e33_720w.jpg)

```
/** 快速移除指定位置处任务 */
void quickRemove(int i) {
    assert i <= size;  //assert生效需要编译器开启断言功能
    
    //指定位置元素直接用最后元素代替,不需要向下调整
    queue[i] = queue[size];
    queue[size--] = null;
}
```

![img](https://pic1.zhimg.com/80/v2-d72f08b8ae0fbaaa9a3da508f9e64b54_720w.jpg)

```
/** 清空任务队列 */
void clear() {
    for (int i=1; i<=size; i++)
        queue[i] = null;
    size = 0;
}
```

> 重新调度任务

重新调度任务不删除堆顶任务，而是将堆顶任务的nextExecutionTime加上period后得到新的nextExecutionTime值，然后根据任务优先级向下调整。

```
void rescheduleMin(long newTime) {
    queue[1].nextExecutionTime = newTime;
    fixDown(1);
}
```

![img](https://pic3.zhimg.com/80/v2-26395bd09d3fc3d4fd4990a3707aa34a_720w.jpg)

> 基础算法

任务队列是优先级队列，基于顺序结构完全二叉树实现的小顶堆。优先级的依据是任务下次执行时间。

![img](https://pic2.zhimg.com/80/v2-7dbfc6a56603dfc301213dd0ba8cfa0d_720w.jpg)

```
/** 提升优先级 */
private void fixUp(int k) {
    while (k > 1) {
        //父结点位置
        int j = k >> 1;
        //如果父结点的下次任务执行时间小于当前结点下次任务执行时间,结束调整操作
        if (queue[j].nextExecutionTime <= queue[k].nextExecutionTime) {
            break;
        }

        //调整任务在任务队列中的位置
        TimerTask tmp = queue[j];  
        queue[j] = queue[k]; 
        queue[k] = tmp;
        k = j;
    }
}
```

```
/** 降低优先级 */
private void fixDown(int k) {
    int j;
    while ((j = k << 1) <= size && j > 0) {
        //选择左右两侧子结点,选择更小的交换位置
        if (j < size && 
            queue[j].nextExecutionTime > queue[j+1].nextExecutionTime) {
            j++; 
        }
        if (queue[k].nextExecutionTime <= queue[j].nextExecutionTime) {
            break;
        }

        //调整任务在任务队列中的位置
        TimerTask tmp = queue[j];  
        queue[j] = queue[k]; 
        queue[k] = tmp;
        k = j;
    }
}
```

调整当前完全二叉树为最小堆。

```
/** 堆化 */
void heapify() {
    for (int i = size/2; i >= 1; i--) {
        fixDown(i);
    }
}
```

## **定时器**

一个定时器内部维护一个任务队列和一个定时线程。在Main线程往任务队列注入任务后，由定时线程异步轮询处理任务队列，这种处理方式实质上是异步串行方式，任务处理并发度为1。

```
public class Timer {

    /** 任务队列 */
    private final TaskQueue queue = new TaskQueue();

    /** 定时线程 */
    private final TimerThread thread = new TimerThread(queue);
}
```

> 构造器

新建Timer实例，同时也新建了任务队列和定时线程，并启动定时线程。启动定时线程前可指定定时线程的名称，以及指定为后台线程。

```
public Timer() {
    this("Timer-" + serialNumber());
}
public Timer(boolean isDaemon) {
    this("Timer-" + serialNumber(), isDaemon);
}
public Timer(String name) {
    thread.setName(name);
    thread.start();
}
public Timer(String name, boolean isDaemon) {
    thread.setName(name); 
    thread.setDaemon(isDaemon);
    thread.start();
}

//单机序列号生成
private final static AtomicInteger nextSerialNumber = new AtomicInteger(0);
private static int serialNumber() {
    return nextSerialNumber.getAndIncrement();
}
```

> 定间隔调度

```
/** 延迟调度 */
public void schedule(TimerTask task, long delay) {
    if (delay < 0)
        throw new IllegalArgumentException("Negative delay.");

    //从当前时间开始延时delay毫秒后调度
    sched(task, System.currentTimeMillis()+delay, 0);
}

/** 定时调度 */
public void schedule(TimerTask task, Date time) {

    //从指定时刻出开始调度
    sched(task, time.getTime(), 0);
}

/** 延时周期性调度 */
public void schedule(TimerTask task, long delay, long period) {
    if (delay < 0)
        throw new IllegalArgumentException("Negative delay.");
    if (period <= 0)
        throw new IllegalArgumentException("Non-positive period.");
    sched(task, System.currentTimeMillis()+delay, -period);
}

/** 定时周期性调度 */
public void schedule(TimerTask task, Date firstTime, long period) {
    if (period <= 0)
        throw new IllegalArgumentException("Non-positive period.");
    sched(task, firstTime.getTime(), -period);
}
```

Timer.schedule()侧重period时间的一致性，保证执行任务的间隔时间相同。

![img](https://pic3.zhimg.com/80/v2-367f6ca013b337ab1d2f2547ed871766_720w.png)

> 定频率调度

```
/** 延时周期性定速调度 */
public void scheduleAtFixedRate(TimerTask task, long delay, long period) {
    if (delay < 0)
        throw new IllegalArgumentException("Negative delay.");
    if (period <= 0)
        throw new IllegalArgumentException("Non-positive period.");
    sched(task, System.currentTimeMillis()+delay, period);
}

/** 定时周期性定速调度 */
public void scheduleAtFixedRate(TimerTask task, Date firstTime, long period) {
    if (period <= 0)
        throw new IllegalArgumentException("Non-positive period.");
    sched(task, firstTime.getTime(), period);
}
```

Timer.scheduleAtFixedRate()侧重执行频率的一致性，任务执行时间加period时间的和相等。

![img](https://pic4.zhimg.com/80/v2-493048111335ad7f57c1f51a29b37753_720w.png)

> 核心调度算法

```
private void sched(TimerTask task, long time, long period) {
    if (time < 0)
        throw new IllegalArgumentException("Illegal execution time.");      
    if (Math.abs(period) > (Long.MAX_VALUE >> 1))
        period >>= 1;

    synchronized(queue) {

        //保证定时器未被取消
        if (!thread.newTasksMayBeScheduled) {
            throw new IllegalStateException("Timer already cancelled.");
        }

        synchronized(task.lock) {
            //保证任务最初处于未使用状态
            if (task.state != TimerTask.VIRGIN) {
                throw new IllegalStateException(
                    "Task already scheduled or cancelled");
            }

            //下次任务执行时间
            task.nextExecutionTime = time;
            //任务执行周期
            task.period = period;
            //设置任务状态为已调度
            task.state = TimerTask.SCHEDULED;
        }

        //往任务队列中添加任务
        queue.add(task);

        //如果队列中该任务为最近要执行的任务,则立即唤醒定时线程处理
        if (queue.getMin() == task) {
            queue.notify();
        }
    }
}
```

> 撤销定时器

```
public void cancel() {
    synchronized(queue) {
        //撤销定时器
        thread.newTasksMayBeScheduled = false;
        //清空任务队列
        queue.clear();
        //唤醒定时线程
        queue.notify();
    }
}
```

> 清理取消状态的任务

```
public int purge() {
    //从队列中移除的任务数
    int result = 0;
    synchronized(queue) {
        for (int i = queue.size(); i > 0; i--) {
            //从队列中移除取消状态任务
            if (queue.get(i).state == TimerTask.CANCELLED) {
                queue.quickRemove(i);
                result++;
            }
        }
        //如果仍有非取消任务,队列重新堆化
        if (result != 0)
            queue.heapify();
    }
    return result;
}
```

## **总结**

读完源码后总结如下，

> 数据结构

小顶堆实现优先级队列，优先级标准是任务下次执行时间。

> 任务状态转换

![img](https://pic3.zhimg.com/80/v2-ad4c978cc45c7a22464335345f525932_720w.jpg)

> 定时器架构图

![img](https://pic3.zhimg.com/80/v2-08da55ef92a07ae0f90a07f18521bb8e_720w.jpg)

> 架构缺陷

单线程串行消费任务，前置任务消费延迟或失败会直接影响后续任务的消费。如果消费前置任务时抛出异常，线程退出，队列中的任务无法被继续消费，定时器失效。