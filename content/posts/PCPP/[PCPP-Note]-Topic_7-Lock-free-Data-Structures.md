---
title: "[PCPP Note] Topic_7 Lock Free Data Structures"
date: 2024-01-03T17:03:28+01:00
draft: true
---

# Topic 7: Lock Free Data Structures

> Goal: \
> Define and motivate lock-free data structures.\
> Explain how compare-and-swap (CAS) operations can be used to solve concurrency problems.

## Compare-And-Swap
* HW support for atomic compound operations
    * Modern processors provide special instructions to for managing concurrent access to shared variables (compare-and-swap (CAS))

* At the programming language level, most languages support an operation: `val.compareAndSwap(a,b)`
    * Compares the value of val and a, and, if they are equal val is set to b, therwise it does nothing. In either case, it returns the current value in val.

Progress conditions for non-blocking DS
1. Wait-free: A method of an object implementation is wait-free if every call finishes its execution in 
a finite number of steps

        • My operations are guaranteed to complete in a bounded number of steps (no matter what other threads do)
2. Lock-free: A method of an object implementation is lock-free if executing the method guarantees 
that some method call (including concurrent) finishes in a finite number of steps

        • Somebody’s operations are guaranteed to complete in a bounded number of my steps
        • Most non-blocking data structures are lock-free
        * 用cas锁的都是，因为如果当前执行cas的线程失败了，证明一定有线程成功了。
3. Obstruction-free: A method of an object implementation is obstruction-free if, from any point after which it executes in isolation, it finishes in a finite number of steps;
(有很多线程在运行，中止了所有只剩一个，这个可以在有限结束)

        • My operations are guaranteed to complete in a bounded of steps (if I get to execute them)

    𝑤𝑎𝑖𝑡𝑓𝑟𝑒𝑒 ⇒ 𝑙𝑜𝑐𝑘𝑓𝑟𝑒𝑒 ⇒ 𝑜𝑏𝑠𝑡𝑟𝑢𝑐𝑡𝑖𝑜𝑛𝑓𝑟𝑒𝑒

<br>
* Why non-blocking ≠ busy-wait?

        1. Threads cannot wait forever because other thread finished incorrectly (这里应该是can吧)
            • Even in obstruction-free, completion must be guaranteed when the thread runs in isolation
        2. All progress definitions for non-blocking programs are part of an algorithm composed by multiple threads 
            • Threads collaborate with each other

<br>
<br>

**CAS implementation in Java**

In Java, CAS and similar operations are accessed through AtomicXX objects
* These classes support atomic lock-free operations for single variables
* AtomicXX CAS operations have the same memory semantics as volatile read/write

* What if we have to update multiple fields?
    * Specification: For any concurrent 
execution of setLower() and setUpper(), it must always hold that lower ≤ upper
    ```java
    public class NumberRange {
        private final AtomicInteger lower = new AtomicInteger(0);
        private final AtomicInteger upper = new AtomicInteger(0);

        public int getLower() { return lower.get();}
        public int getUpper() { return upper.get();}

        public void setLower(int i) {
            if (i > getUpper()) {
                throw new IllegalArgumentException("Can't set lower to " + i +" > upper");
            }
            lower.set(i); 
        }
        public void setUpper(int i) {
            if (i < getLower()) {
                throw new IllegalArgumentException(("Can't set upper to " + i +" < lower");
            }
            upper.set(i); 
        }
    }
    // Assume current [0,10], two thread, setLower=5, setUpper=4
    ```
    ===> FIX
    ```java    
    private static class IntPair {
        private final int lower;
        private final int upper;
        public IntPair(int lower, int upper) {
            this.lower = lower;
            this.upper = upper;
        }
    }

    public class CasNumberRange {
        private final AtomicReference<IntPair> values =
        new AtomicReference<IntPair>(new IntPair(0,0));
        
        public int getLower() { return values.get().lower;}
        public int getUpper() { return values.get().upper;}

        public void setLower(int i) {
            while(true) {
                IntPair oldv = values.get();
                if (i > oldv.upper) {
                    throw new IllegalArgumentException(
                "Can't set lower to " + i +" > upper");
            }
            IntPair newv = new IntPair(i, oldv.upper);
            if(values.compareAndSet(oldv,newv))
                return;
        }
    }
    … // Similarly for setUpper();
    }

    ```
**A note on correctness and thread-safety**

Operations on AtomicXX are not explicitly part of the Java memory model
* We cannot use them to reason about correctness or thread-safety
* LATER, we will see a finer-grained definition of correctness for concurrent objects, Linearizabilits


<br>

## CAS based Lock implementation
1. Simple TryLock
* Non-blocking tryLock, the lock may be acquired only once
* Regular unlock
    ```java
    class SimpleTryLock {
        // Refers to holding thread, null iff unheld
        private final AtomicReference<Thread> holder = new AtomicReference<Thread>();

        public boolean tryLock() {
            final Thread current = Thread.currentThread();
            return holder.compareAndSet(null, current);
        }
        public void unlock() {
            final Thread current = Thread.currentThread();
            if (!holder.compareAndSet(current, null))
                throw new RuntimeException("Not lock holder");
        }
    }
    ```

2. Reentrant TryLock
* Non-blocking tryLock, the lock may be acquired repeatedly by the same thread
* Regular (reentrant) unlock
    ```java
    class ReentrantTryLock {
    private final AtomicReference<Thread> holder = new AtomicReference<Thread>();

    private volatile int holdCount = 0;

    public boolean tryLock() {
        final Thread current = Thread.currentThread();
        if (holder.get() == current) { // question below 
            holdCount++;
            return true;
        } else if (holder.compareAndSet(null, current)) { 
            holdCount = 1; 
            return true;
        } 
        return false; 
    }

    public void unlock() {
        final Thread current = Thread.currentThread();
        if (holder.get() == current) {
            holdCount--;
            if (holdCount == 0) 
                holder.compareAndSet(current, null);
            return;
        } else
            throw new RuntimeException("Not lock holder");
        }
    }
    ```
<u>Do we have a race condition in this branch of the if-statement?</u> No, only same thread can enter this branch, it can only have sequential execution.

3. Simple Lock
* Blocking lock, the lock may be acquired only once
* Regular unlock
```java
class SimpleLock {
    private final AtomicReference<Thread> holder = new AtomicReference<Thread>();
    // The FIFO queue of threads waiting for this lock
    private final Queue<Thread> waiters = new ConcurrentLinkedQueue<Thread>();

    public void lock() {
        final Thread current = Thread.currentThread();
        waiters.add(current);
        while (waiters.peek() != current || !holder.compareAndSet(null, current)) {
            LockSupport.park(this);
    }
        waiters.remove();
    }

    public void unlock() {
        final Thread current = Thread.currentThread();
        if (holder.compareAndSet(current, null)) 
            LockSupport.unpark(waiters.peek()); 
        else
            throw new RuntimeException("Not lock holder");
    }
}
```

## Scalability of CAS
* Pros
    * A CAS operation is faster than acquiring a lock
    * An unsuccessful CAS operation does not cause thread de-scheduling (blocking)
    * 总失败的话应该用锁，这个是乐观机制。
* Cons
    * CAS operations result in high memory overhead

## The ABA Problem
1. Thread 1 starts popping A
2. Before thread 1 finishes, thread 2 pops A and B
3. Thread 2 pushes A back // recovered from memory (same as thread 1 was popping)
4. Thread 1 finishes popping A
    * Incorrectly, as it thinks that A is the same as before thread 2 operations

• It is a memory allocation issue which affects mainly languages 
without garbage collection (e.g., C and C++)\
*  在C++中，我们对指针(类比于JAVA中的引用)采用CAS操作，那么即使两个指针相同，他们也未必引用同一个对象，有可能是第一个指针所指向的内存被释放后，第二个对象恰好分配在相同地址的内存。

• Languages such as Java do not have this problem because 
garbage collection ensures that newly created objects are 
fresh
* Step 3 in the previous step would have created a new A 
object

## Lock-free data structures: 
Queue / Stack (see next note)