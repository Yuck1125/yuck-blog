---
title: Callable异常处理分析
date: 2019-07-10 11:28:24
tags: ["java","ThreadPoolExecutor"]
categories: "技术"
---
### 前言
&emsp;&emsp;分析前几天遇到的一个老代码留下的坑。线程池中运行`Callable`线程时抛出的异常捕获不到，简化的逻辑如图,环境是jdk8：
![](/img/catch_callable_exception-1.png)
运行结果：
![](/img/catch_callable_exception-2.png)
### 解决方案
1. 线程池返回`Future<>`，调用其`get()`
2. 在Callable中 try-catch可能抛错的异常
![](/img/catch_callable_exception-3.png)
运行结果：
![](/img/catch_callable_exception-4.png)
### 源码分析
&emsp;&emsp;不难发现线程池提交时创建的类为`FutureTask`。
```
   public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }

 protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
```
看`FutureTask.run()`之前，先简单结束一下其关的属性。
* state：线程的状态。主要有如下几种：
   *  NEW： 新建
   *  COMPLETING: 运行在
   *  NORMAL: 正常完成
   *  EXCEPTIONAL: 异常
   *  CANCELLED: 取消
   *  INTERRUPTING: 被中断的中间状态
   *  INTERRUPTED: 被中断的最终状态
* outcome： get()返回值

```
   public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
注意这里
```
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }

   protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }
```
这里线程在运行时抛出异常时，`FutureTask`把异常信息赋值给`outcome`，并将`state`设为`EXCEPTIONAL`。
```
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }

    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```
在调用get()时，如果运行时抛出异常，此时会抛出异常。
### 总结
&emsp;&emsp;这种坑还是代码规范的问题。`Callable`返回结果并没有被使用可以用`Runnable`代替;try-catch代码的习惯。
