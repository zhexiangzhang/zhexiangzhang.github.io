---
title: "[PCPP Note] Topic_4 Testing"
date: 2023-12-22T14:04:42+01:00
type: post
tags: ["PCPP"]
showTableOfContents: true
---

# Topic 4: Testing

> Goal: \
> Explain the challenges in ensuring the correctness of concurrent programs. \
Describe different testing strategies for concurrent programs, and their advantages and disadvantages.

## Program correctness
A (concurrent) program is correct if and only if it satisfies its specification
* Specifications are often stated as a collection of program properties
* Concurrency properties
    1. Safety – “Something bad never happens”
       * Ex: The field size of a collection is never less than 0
    2. Liveness – “Something good will eventually happen”
       * Ex: It should always be possible to eventually add elements to the collection
* Counterexamples
    1. Counterexamples in safety properties <span style="color:yellow;">[we will fouce on this]</span> 
        * A counterexample is a finite interleaving where the property does not hold
            * Ex: The field size of a collection is never less than 0 => insert an element but delete it twice.
    2. Counterexamples in Liveness property
        * A counterexample is an infinite interleaving where the property never holds.
<br><br>

## Testing concurrent programs
* Performance tests (next Topic)
* Functional Correctness tests 
    * Testing concurrent programs is about writing tests to find undesired interleavings <span style="color:DarkTurquoise;">(counterexamples)</span> that violates a property(if any)
    * Since concurrent execution is non-deterministic, it is not guaranteed that tests will trigger undesired interleavings

### Sequential tests in JUnit 5
```java
// Class to test
class CounterDR implements Counter {
    private int count;
    public CounterDR() {
        count = 0;
    }
    public void inc() {
        count++;
    }
    public int get() {
        return count;
    }
}

// Test Class
public class CounterTest {
    
    private Counter count;
    
    @BeforeEach  // executed before each test. It is useful to initialize the objects to test
    public void initialize() { 
    count = new CounterDR();
    }
    
    @RepeatedTest(10)
    @DisplayName("Counter Sequential")
    public void testingCounterSequential() 
    {
        int localSum = 0;
        for (int i = 0; i < 10_000; i++) {
            count.inc();
            localSum++;
        }
        assertTrue(count.get()==localSum);
    }
}
```
### Concurrent Correctness Test in JUnit 5
Some strategies to take into account when developing a test:
1. Precisely define the property you want to test
    * Use assertions to test properties
        ```java
        // “after N threads execute inc() X times, the value of the counter must be equal to N*X”
        Class CounterTest {
            Counter count;
            …
            public void testingCounterParallel(int nrThreads, int N) {
                // body of the test
                assert(N*nrThreads == count.get());
            }
            … 
        }
        ```
2. When test multiple implementations, define an interface for the class that are testing is useful
    ```java
    public interface Counter {
        public void inc();
        public int get();
    }

    // A thread-safe interger class,
    class CounterAto implements Counter {
        private AtomicInteger count;
        public CounterAto() { count = new AtomicInteger(0); }
        public void inc() { count.incrementAndGet(); }
        public int get() { return count.get(); }
    }

    class CounterDR implements Counter {
        private int count;
        ...
    }
    ```
3. Concurrent tests require a setup for starting and running multiple threads
    * Maximize contention to avoid a sequential execution of the threads <span style="color:yellow;">( ie.Maximizing the number of threads running concurrently)</span> 
        * In java, A cyclic barrier may be used to decrease the chance that threads are executed sequentially
            ```java
            class TestCounter {
                // Shared variable for the tests
                CyclicBarrier barrier;
                …

                public void testingCounterParallel(int nrThreads, int N) {
                    …
                    // init barrier
                    barrier = new CyclicBarrier(nrThreads + 1); // We need + 1 for main thread

                    for (int i = 0; i < nrThreads; i++) {
                        new Thread(() -> {
                            barrier.await(); // wait until all threads are ready
                            // thread execution
                            barrier.await(); // wait until all threads are finished
                        }).start();
                    }

                    try {
                        barrier.await();
                        barrier.await();
                    } catch (InterruptedException |  BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                    …
            }
            ```        
            <u>Can we wait until threads finish without using the barrier?</u> Yes. We can use join to wait, but then we need to define a list to store all threads. Barrier is more convenient.
    * You may need to define thread classes
        ```java
        class TestCounter {
            
            // Shared variable for the tests
            CyclicBarrier barrier;
            Counter count;
            …
            
            public void testingCounterParallel(int nrThreads, int N) {
                …
                // init barrier
                barrier = new CyclicBarrier(nrThreads + 1);

                for (int i = 0; i < nrThreads; i++) {
                    new Turnstile(N).start(); // now  we can simply start the thread in the test
                }

                try {
                    barrier.await();
                    barrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
                …
            }

            public class Turnstile extends Thread {
                
                private final int N;
                
                public Turnstile(int N) { this.N = N; }
                
                public void run() {
                    try {
                        barrier.await();
                        for (int i = 0; i < N; i++) {
                            count.inc();
                        }
                        barrier.await();
                    } catch (InterruptedException | BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            }
            …
        }
        ```  

4. Run the tests multiple times and with different setups to try to maximize 
the number of interleavings tested
    * some interleavings are difficult to trigger
    * use @RepeatedTest() in Junit

<br><br>
## Testing a Bounded Buffer
* Goal: test a functional correctness property of a bounded buffer that may be accessed by producers and consumers concurrently
* Definition:
    * Producers may put elements in the buffer as long as there is space. <span style="color:yellow;">Otherwise, they must wait.</span> 
    * Consumers can take elements from the buffer as long as it is not empty. <span style="color:yellow;">Otherwise, they must wait.</span> 
    * The buffer is implemented as a **circular** buffer
    * Synchronization is implemented using a monitor

* Implementation:
    * Bounded buffer implementation using a monitor
    * It uses two conditions variables for threads to wait:
        * notFull – wait until buffer is not full
        * notEmpty – wait until buffer is not empty
        ```java
        private final E[] items;
        private int putPtr, takePtr, numElems;
        private final Lock lock;
        private final Condition notFull;
        private final Condition notEmpty;
        ```     

        ```java
        public void put(E element) {
            lock.lock();
            try {
                while(numElems >= items.length) 
                    notFull.await();
                doInsert(element);
                numElems++;
                notEmpty.signalAll();
                …
            } finally { lock.unlock();}
        }

        private void doInsert(E element) {
            items[putPtr] = element;
            if (++putPtr == items.length) putPtr = 0;
        }
        ```   

        ```java
        public E take() {
            lock.lock();
            try {
                while(numElems <= 0)
                    notEmpty.await();
                E result = doTake();
                numElems--;
                notFull.signalAll();
                return result;
                …
            } finally { lock.unlock(); }
        }

        private E doTake() {
            E result = items[takePtr];
            items[takePtr] = null;
            if (++takePtr == items.length) takePtr = 0;
            return result;
        }
        ```   
    * Property to check
        * If several threads put and take the same number of 
elements, the sum of the put elements and the sum of the taken elements 
must be equal

        ```java
        // more code details: see slide
        ```   

<br><br>
## Deadlocks
A deadlock occurs when all threads are waiting for a lock held by 
another threads 
* EG. Dinning philosophers
    * Philosophers only think and eat
    * A philosopher must pick both left and right forks to start eating
    * If all philosophers grab their right fork, we reach a deadlock state. Note that this is behaviour is captured by a finite interleaving.
* Testing for deadlocks is complicated and often not possible
    * Reason (from GPT):
        * Vast State Space: Concurrent programs can have an enormous number of possible states, making it impractical to test all possible interleavings.
        * Non-Determinism: the interleavings can vary, making deadlocks unpredictable and hard to reproduce.
    * We should define a maximum duration for an operation, after 
which, we deem the execution as deadlocked
    * If, when a running a test, we observe that the program does
not terminate for a long time, it might be due to deadlocks
* <u>Are deadlocks a safety or liveness property?</u>   
    * Safety property
    * (definition) the "bad thing" is a deadlock, which will never happen. 
    * (definition of counterExample) The occurrence of a deadlock means that a specific undesirable state has been reached – a clear violation of a safety property happen in finite interleavings. 
       

<br><br>
## Formal Verification
* **Limitations of testing**
    * Testing cannot be used to prove the absence bugs
    * Tests can be seen as interleaving generators. For most systems, it is virtually impossible to write a set of tests that cover all possible interleavings in the system.
* **Formal Verification**
    * Formal verification aims to prove that a program satisfy a specification (properties)
    * It treats programs and properties as mathematical objects AND can prove that programs satisfy their specifications
        * Manually
        * Automatically (EG. model-checking)
            * Model-checking transforms programs into a finite-state models that encapsulate all possible interleavings in the system
            * But: even for small programs the computational cost by using Model-checking to prove the system satisfies its specification can be astronomically expensive






