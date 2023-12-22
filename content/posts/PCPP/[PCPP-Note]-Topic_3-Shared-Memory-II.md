---
title: "[PCPP Note] Topic_3 Shared Memory II"
date: 2023-12-20T01:19:50+01:00
draft: true
---
# Topic 3: Shared Memory II

> Goal: \
> Define and explain what makes a class thread-safe. \
> Explain the issues that may make classes not thread-safe. 

## Thread-safe

**Thread-safe classes**
* A <span style="color:YELLOW;">class</span> is said to be <span style="color:YELLOW;">thread-safe</span> if and only if no concurrent execution of method calls or field accesses (read/write) result in race conditions.

**Thread-safe program**
* A concurrent <span style="color:YELLOW;">program</span> is said to be <span style="color:YELLOW;">thread-safe</span> if and only if it is race condition free

It is important to note that: <span style="color:DarkTurquoise;">For any program p, p only accesses thread-safe classes ⇏ p is a thread-safe programs.</span>  (Programs using thread-safe classes 
may contain race conditions.)

<u>Does this hold? p is a thread-safe program ⇏ p only accesses thread-safe classe</u>
Yes. A way to read the implication is: “Can we have thread-safe programs if we are not using thread-safe classes?” EG. programs that do not work on shared memory, or only read data (from non thread-safe classes). Then you may use a non-thread safe class in a thread safe manner. In these cases, you have a thread safe program without using thread-safe classes.

## Thread-safe classes 
* To analyse whether a class is thread-safe, we must identify/consider:
    1. Identify the class state
    2. Make sure that mutable class state does not escape
    3. Ensure safe publication
    4. Whenever possible define class state as immutable
    5. If class state must be mutable, ensure mutual exclusion<br><br>

    * **<span style="color:DarkTurquoise;">Class state：</span>** 
        * <u>If a class has no state (variables), is it thread-safe?</u> Assuming that all methods are pure
(the output only depends on the input), then yes. This is because there is no shared state.
    * **<span style="color:DarkTurquoise;">Escaping：</span>** 
        * Defining all (shared) class state variables as private to ensures they will only be accessed through public methods.
            ```java
            class Counter {
                // class state (variables)
                int i=0;
                // class methods
                public synchronized void inc(){i++;}
            }

            // program using Counter
            Counter c = new Counter();
            new Thread(() -> { c.inc(); }).start();

            // escaped the lock in inc() 
            new Thread(() -> { c.i++; }).start();
            ```
             <u>Is the class Counter thread-safe?</u>
        No, even though the inc method is protected by a lock, variable i is public. Consequently, it may be accessed by many threads without being protected by a lock => allow several threads in the critical section.<br><br>
            ```java
            class IntArrayList {
                // class state
                private List<Integer> a = new ArrayList<Integer>();
                public synchronized void set(Integer index, Integer elem) { 
                    a.set(index,elem); 
                }
                public synchronized List<Integer> get() { 
                    return a; 
                }
            }

            IntArrayList array = new IntArrayList();
            // access state with lock
            new Thread(() -> { array.set(0,1); }).start();
            // access state without locks
            new Thread(() -> { array.get().set(0,42); }).start();
            ```
            <u> Is the class IntArrayList thread-safe?</u> when a method returns an object, we get a reference to that object. Therefore, even if obtain the reference using locks, later we can modify the content of the object without locks.</u><br>
            <u> Is this program thread-safe?</u> No, because the value of array[0] after the execution of IntArrayList.set(0,1) may be equal 42. Note that the array.get().set() uses no lock, so the locking and unlocking in IntArrayList.set does not prevent the data race.

    * **<span style="color:DarkTurquoise;">Safe publication：</span>** 
        * ensure initialization happens-before publication
            * That is, before making accessible a reference to an object, all its fields must be correctly initialized
            ```java
            public class UnsafeLazyInitialization {
                private static Resource resource;
                public static Resource getInstance() {
                    if (resource == null)
                        resource = new Resource();
                    return resource;
                }
            }
            ```
            <u>Is this class thread-safe?</u>
            No, if the expected behaviour of getInstance is to create a Resource object only once. Two threads may run getInstance() at the same time, both get resource==null and both creating an 
instance of Resource. With the last thread’s instance being the value that stays in resource (state of the class).<br><br>
        * **Object initialization & visibility**
            * Visibility issues may appear during initialization of objects
                ```java
                public class UnsafeInitialization {
                    private int x;
                    private Object o;
                    public UnsafeInitialization() {
                        x = 42;
                        o = new Object();
                    }
                }
                ```
                For the thread executing the constructor, there are no visibility issues, but if a reference to an instance of UnsafeInitialization object is accessible to another thread, it might not see x==42 or o completely initialized
            * We can address visibility issues during initialization as follows
                ```java
                public class UnsafeInitialization {
                    private volatile int x;
                    private final Object o;
                    public UnsafeInitialization() {
                        x = 42;
                        o = new Object();
                    }
                }            
                ```
                <u>Why do these solutions solve visibility issues?</u> For x, because volatile variables are flushed to main memory for every write (so all threads read the same values). For o, because the JVM ensures visibility of complex objects declared as final.
                | For primitive types, we can:   | For complex objects, we can:     | 
                | --------  | -------- | 
                | Declare them as <span style="color:yellow;">volatile</span> | Declare them as <span style="color:yellow;">final</span>| 
                | Declare them as <span style="color:yellow;">final</span> (only works if the content is never modified) | Initialize as the default value: <span style="color:yellow;">null</span>. (only works if the default value is acceptable) |
                | Initialize as the default value: <span style="color:yellow;">0</span>. (only works if the default value is acceptable) | Use the <span style="color:yellow;">AtomicReference</span> class | 
                | Use corresponding atomic class from Java standard library:  <span style="color:yellow;">AtomicInteger </span>|  | 
                |   |  | 

                [] The previous solutions ensure safe publication because:
                * They established a happens-before relation between initialization and access the object’s reference (publication)
                  * – A write to a volatile field happens-before every subsequent read of that field.
                  * – The default initialization (zero, false, or null) of any object happens-before any other actions of a program.
                  * – The initialization of a final field happens-before any other actions of a program (after the constructor has finished its execution)
                    * If the constructor of the class leaks a reference of the object being constructed before it has completed its execution, then there is no happen before relation with the accesses to final field.
                * At the JVM level, the reason is that:
                  * – final fields cannot be cached or reordered during initialization
                  * All fields are initialized with default values during class loading
                  * writes on volatile are flushed to main memory and reordered (during initialization)


    * **<span style="color:DarkTurquoise;">Immutability：</span>** 
        * An immutable object is one whose state cannot be changed after initialization
            * think of it as a constant
            * The final keyword in Java prevents modification of fields            
        * A immutable class is one whose instances are immutable objects
            * <u>Are immutable classes thread-safe?</u> If they are safely published (i.e., without visibility issues during publication), then yes. This is because fields can only be read.
            * <u>Does defining all fields as final ensure that the class is immutable?</u> No, **complex objects** may still be updated, e.g., we can add elements to a list declared as final.
            * <u>If in a class, no fields are defined as final, is it possible to make it immutable?</u> Yes, if we ensure that those fields cannot be accessed, and have methods that only read them.
        * To ensure thread-safety of immutable classes you **simply** need to make sure:
            * Objects are safely published (==Since immutable objects do not change the state after initialization, data races can only occur during initialization==)
            * No fields can be modified after publication
            * Access to inner mutable object do not escape
                ```java
                public final class ThreeStooges {
                    private final Set<String> stooges = new HashSet<String>();
                    public ThreeStooges () {
                        stooges.add(“Moe”);
                        stooges.add(“Larry”);                    
                    }
                    public Boolean isStooge(String name) {
                        return stooges.contains(name)
                    }
                }
                ```
                <u>Why is this class thread-safe? (tip: there are 3 main reasons)</u>

                    * 1. The state (stooges) is declared as final so it is safely published.    
                    * 2. The state is declared as private and no method returns a reference to stooges, so there is no way to access the field other than using the method isStooge()
                    * 3. The method isStooge() does not modify the state, only reads it

    * **<span style="color:DarkTurquoise;">Mutual exclusion：</span>** 
      * Whenever shared **mutable** state is accessed by several threads is must be protected by locks
        * <u>Are Monitors a thread-safe class?</u> Yes, if implemented according to the specification in Note 2, and they are safely published.
            *  A monitor consists of: 
                1. Internal state (data)
                2. Methods (procedures)<br>
                    – All methods in a monitor are mutually exclusive (ensured via locks)<br>
                    – Methods can only access internal state
                3. Condition variables (or simply conditions)    
    
## Other synchronization primitives (synchronizers)
### Semaphores
* Semaphores are synchronization primitives that allow at most c
number of threads in the critical section where c is called the 
capacity [ <span style="color:yellow;">typically used to control the number of threads accessing a resource </span>]
* A semaphore consists of:
    1. An integer capacity (c) – Initial number of threads allowed in the critical section
    2. A method acquire() <br>
       * Checks if c > 0, if so, it decrements capacity by one (c--) and allows the calling thread to make progress, otherwise it blocks the thread<br>
       * It is a blocking call
    3. A method release()
       * It checks whether there are waiting threads, if so, it wakes up one of them, otherwise it 
increases the capacity by one (c++)
       * It is non-blocking
        ```java
        ReadWriteMonitor m = new ReadWriteMonitor();
        Semaphore semReaders = new Semaphore(5,true);
        Semaphore semWriters = new Semaphore(5,true);
        for (int i = 0; i < 10; i++) {
            // start a reader
            new Thread(() -> {
                semReaders.acquire();
                m.readLock();
                // read
                m.readUnlock();
                semReaders.release();
            }).start();
            
            // start a writer
            new Thread(() -> {
                semWriters.acquire();
                m.writeLock();  // the semaphore make no difference for writers as the monitor only allows only one writer at a time.
                // write
                m.writeUnlock();
                semWriters.release();
            }).start(); 
        }

        // We can also impose this constraint(number of readers/writers) by implementing monitor condition instead of need a semaphore
        ```
* <u>If we set the capacity of a semaphore to 1, does it behave like a lock?</u> Depends on the behaviour of release. If release increases the capacity of the semaphore when c==1, then no. But if the semaphore does not increase capacity when c==1, then yes.

### Barriers
* Barriers are synchronization primitives used to wait until several thread reach some point in their computation
* Barriers consists of:
    1. A number parties to wait for
    2. A method await() <br>
       * If the number of waiting threads is less than parties, then the calling thread blocks, otherwise all waiting threads wake up and the calling thread is allowed to make progress

        ```java
        // Several threads are used to initialize an array (each a different position), the barrier is used for threads to know when the initialization is finished

        int parties = 10;
        CyclicBarrier cb = new CyclicBarrier(parties);
        int[] shared_array = new int[parties];
        …
        for (int i = 0; i < parties; i++) {
            new SetterClass(i).start();
        }
        …
        public class SetterClass extends Thread {
            int index;
            public SetterClass(int index) {this.index = index;}
            public void run() {
                shared_array[index] = index+1;
                cb.await();
                // After this point the array is initialized and it is safe to read it
            }
        }
        ```