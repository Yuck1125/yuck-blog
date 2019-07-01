---
title: LongAdder源码分析
date: 2019-06-25 14:51:15
tags: "java"
categories: "技术"
---
### Intro
&emsp;&emsp;JDK8 在并发工具包下增加了`LongAdder、DoubleAdder`类，提供原子的增减功能。本文主要介绍一下`LongAdder`，根据`Doug Lea`的文档描述,该类在高并发的情况下，吞吐量会比`AtomicLong`高很多，当然会牺牲一定的空间。
### AtomicLong
&emsp;&emsp;JDK8以前JUC下面的原子类都是通过Unsafe类提供CAS的能力来实现的，而Unsafe类是由C来调用硬件级的原子指令实现的。AtmicLong的部分代码如下：
```
 // setup to use Unsafe.compareAndSwapLong for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
            (AtomicLong.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    
public final long addAndGet(long delta) {
        return unsafe.getAndAddLong(this, valueOffset, delta) + delta;
    }
```
Unsafe类部分代码：
```
 public final long getAndAddLong(Object var1, long var2, long var4) {
        long var6;
        do {
            var6 = this.getLongVolatile(var1, var2);
        } while(!this.compareAndSwapLong(var1, var2, var6, var6 + var4));
        return var6;
    }
```
`getLongVolatile`方法是从内存中取到对应的值，然后尝试CAS更新，成功就返回,失败就不断重试直到成功。在高并发的情况下，会造成很大的开销。
### LongAdder
&emsp;&emsp;先看一下该类的结构;  
![](/img/Adder-1.png)  
LongAdder继承自Striped64,其核心功能基本都是`Striped64`实现的。介绍一下`Striped64`的主要属性
* 基础值`base`
* 内部类`Cell`
```
    @sun.misc.Contended static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```
`@sun.misc.Contended`注解，可以[解决伪共享的问题](https://www.jianshu.com/p/c3c108c3dcfd)

* 数组`Cells`,数组大小为2的N次方,首次初始化为2
* `NCPU` 控制数组的最大为cpu的核数


#### 分析
##### add方法
```
  public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        // 第一次add() 直接调用casBase()。
        //成功就结束，否则走到下面逻辑
        if ((as = cells) != null || !casBase(b = base, b + x)) {
             // 设置竞争标识。true->目前没有竞争
            boolean uncontended = true;
            // as数组为空或者size为0
            // 或者当前线程所分配的Cell为空(as数组大小为2的N次方，所以这里其实就是取模的操作a%b==a&(b-1))
            // 或者cas更新Cell失败
            //就执行longAccumulate()方法
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }
 
   final boolean casBase(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
    }
```
##### longAccumulate方法  
&emsp;&emsp;先看一下longAccumulate方法大概逻辑;  
![](/img/Adder-2.png)  

* 首先获取当前线程的`probe` 如果没有初始化就初始化并设置`wasUncontended==true`
* 循环中的逻辑有三个大分支 
    1. cells数组内有元素
    2. 尝试扩容
    3. 尝试用casBase()计算值

```
        int h;
        if ((h = getProbe()) == 0) {
            //初始化当前线程的prob
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            // cells数组内有元素
            if ((as = cells) != null && (n = as.length) > 0) {
                // 当前线程分配到的桶位 值为null 就对其赋值
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        // 这里casCellsBusy()防止了并发的安全问题
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                // 重新确认是否已赋值
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            // false 说明现在又线程在进行初始操作，自旋检查created标识 
                            if (created)
                                break;
                            continue;        
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                    // Cell尝试cas操作
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                    // 扩容到最大容量 或者引用过期
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                    // false-> 重新自旋 ps.个人觉得是防止引用过期,然后再重试下 
                else if (!collide)
                    collide = true;
                    // 扩容，扩大两倍
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
            }
            // 尝试扩容
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        // 根据奇偶分配
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            // 尝试累加值
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
        
       final boolean casCellsBusy() {
        return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);
    }
```

### 总结
&emsp;&emsp;LongAdder的性能毋容置疑，主要的缺点就是它不能获取自增后的更新值。
