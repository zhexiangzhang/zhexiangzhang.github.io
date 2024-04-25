---
title: "[PCPP Note] Topic_5 Performance Measurements"
date: 2023-12-22T19:15:20+01:00
type: post
tags: ["PCPP"]
showTableOfContents: true
---

# Topic 5: Performance Measurements

> Goal: \
> Motivate and explain how to measure the performance of Java code. \
> Illustrate some of the pitfalls there are in doing such measurements.

## Performance measurements: motivation and introduction
* Motivations for Concurrency
  * Exploitation: Hardware capable of simultaneously executing multiple streams of statements
  * Motivation 1: Time consuming computations 
    * Searching in a (large) text   
    * Computing prime numbers
  * Motivation 2: Analyzing code 
    * Thread creation is expensive ? But how expensive => get such numbers

    
## Pitfalls (and avoiding them)
measuring a (simple) function
Results will usually vary, mean and variance
## Calculating means and variance (efficiently)
```java
public static double Mark5() {
    int n = 10, count = 1, totalCount = 0;
    double dummy = 0.0, runningTime = 0.0, st = 0.0, sst = 0.0;
    do {
        count *= 2;
        st = sst = 0.0;
        for (int j=0; j<n; j++) {
            Timer t = new Timer();
            for (int i=0; i<count; i++)
                dummy += multiply(i); // to confuse compiler to not optimize
            runningTime = t.check();
            double time = runningTime * 1e9 / count;
            st += time;
            sst += time * time;
            totalCount += count;
        }
        double mean = st/n, sdev = Math.sqrt((sst - mean*mean*n)/(n-1));
        System.out.printf("%6.1f ns +/- %8.2f %10d%n", mean, sdev, count);
    } while (runningTime < 0.25 && count < Integer.MAX_VALUE/2);
    return dummy / totalCount;
}

```
## Measurements of thread overhead
## Algorithms for parallel computing
```
Parameterizing function to be measured
```java
public static double Mark6(String msg, IntToDoubleFunction f) {
    ...
    do {
        ...
        for (int j=0; j<n; j++) {
            Timer t = new Timer();
            for (int i=0; i<count; i++)
                dummy += f.applyAsDouble(i);
            ...
        }
        double mean = st/n, sdev = Math.sqrt((sst - mean*mean*n)/(n-1));
        System.out.printf("%6.1f ns +/- %8.2f %10d%n", mean, sdev, count);
    } while (runningTime < 0.25 && count < Integer.MAX_VALUE/2);
    return dummy / totalCount;

    public interface IntToDoubleFunction { double applyAsDouble(int i); }
    
    Mark6("multiply", i -> multiply(i));
    }
```
## Measurements of thread overhead
Thread creation 
```java
// Takes 700 ns
Mark7("Thread create", 
    i -> {
        Thread t = new Thread(() -> {
            for (int j=0; j<1000; j++) 
              ai.getAndIncrement(); 
    });
    return t.hashCode(); // to confuse compiler to not optimize
});
```
<u>What are we really measuring?</u>  Thread creation (not include exectution)

Thread create + start
```java
// Takes ~ 47000 ns
Mark6("Thread create start", 
    i -> {
        Thread t = new Thread(() -> {
            for (int j=0; j<1000; j++)
                ai.getAndIncrement(); 
        });
        t.start();
        return t.hashCode();
});
```
* <u>What are we really measuring?</u> Thread create + start. Note: does not include executing the loop (we didnt use join to wait)


* a lot of work goes into starting a thread, even after creating it
* Never create threads for small computations !!!
<br><br>
## Algorithms for parallel computing
Multithreaded version of CountPrimes 

* thread0: 2,3,4,5,...
thread1: ...........
...
...
threadN: ...........

```java
private static long countParallelN(int range, int threadCount) {
    final int perThread= range / threadCount;
    final LongCounter lc= new LongCounter();
    Thread[] threads= new Thread[threadCount];

    for (int t=0; t<threadCount; t++) {
        final int from= perThread * t, 
            to= (t+1==threadCount) ? range : perThread * (t+1); 
        threads[t]= new Thread(()
            - > {for (int i=from; i<to; i++) 
                        if (isPrim(i)) lc.increment();
        });
    }

    for (int t=0; t<threadCount; t++) threads[t].start();
    try { for (int t=0; t<threadCount; t++) threads[t].join();
    } catch (InterruptedException exn) { }
    return lc.get();

    /*
        countSequential 5922958.0 ns 289879.33 
        countParallel 1 7107236.6 ns 448417.55 
        countParallel 2 6069944.7 ns 802224.61 
        countParallel 3 3621185.5 ns 152693.03 
        countParallel 4 3124067.0 ns 640480.51 
        countParallel 5 3699514.7 ns 364428.77 
        countParallel 6 4114074.2 ns 642562.19 
        countParallel 7 2049595.7 ns 26888.15 
        countParallel 8 1801465.6 ns 12532.85 
        countParallel 9 1793099.1 ns 11017.57 
        countParallel 10 1798921.4 ns 11541.43 
        countParallel 11 1807408.3 ns 9763.61
    */
 }
```
* <u>Is this good or bad, and why?</u> 
not fair (difficulty is not same), threads performing simple tasks will finish earlier than bottleneck ones. 

<br><br>
<b>Breaking the task into smaller pieces/tasks</b><br>
* When a thread is done with one task, it gets a new task until all tasks are done
* Threads are expensive, use threadPool to resue threads
```java
public class countPrimesTask implements Runnable {
    private final int low;
    private final int high;
    private final ExecutorService pool;
    
    @Override public void run() {
        int mid= low+(high-low)/2;
        pool.submit( new countPrimesTask(low, mid, pool) );
        pool.submit( new countPrimesTask(mid+1, high, pool) );
    }
}
```
* Shortcomings:
    1. How to stop?
    2. Will create too many "small" tasks
    3. Returning result (# primes)

<br><br>
<b>Reducing the number of tasks</b>
```java
@Override
    public void run() {
        if ((high-low) < threshold) { 
            for (int i=low; i<=high; i++) if (isPrime(i)) lc.increment();
        } else {
            int mid= low+(high-low)/2;
            pool.submit(new countPrimesTask(lc, low, mid, pool, threshold) );
            pool.submit(new countPrimesTask(lc, mid+1, high, pool, threshold) );
    }
}
```

* Shortcomings:
    1. How to stop? (as long as submit the task, it will not wait, the program will end immediately 
        * 这里和之前算法中的递归不相同，之前的递归调用处直到获得结果才返回，但这里是向线程池提交，返回的时候任务不一定执行完)
    3. Returning result (# primes)

* Thread vs Executor
    * Counting primes in the range 2..1_000_000s
        * Sequential 1.2 Sec
        * Threads (4) 0.5 Sec
        * Executor 0.4 Sec

<br>

**Combining tasks**
```java
@Override
    public void run() {
        if ((high-low) < threshold) { 
            for (int i=low; i<=high; i++) if (isPrime(i)) lc.increment();
        } else {
            int mid= low+(high-low)/2;
            Future<?> f1 = pool.submit(new countPrimesTask(lc, low, mid, pool, threshold) );
            Future<?> f2 = pool.submit(new countPrimesTask(lc, mid+1, high, pool, threshold) );

            try { f1.get();f2.get(); }
            catch (InterruptedException | ExecutionException e) { }
    }
}
```
* Shortcomings:

    1. Returning result (# primes)

<u>Does the order of f1.get and f2.get matter?</u>
No. Two tasks run independently, and it's uncertain which one will finish first. Even if Task 2 finishes first, it still has to wait T1.\
<u>How do we get the result # primes ?</u>
do lc++ in if, and get the final result of the global variable lc.
