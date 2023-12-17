---
title: "[PCPP Note] Topic_2 Shared Memory I"
date: 2023-12-17T23:53:26+01:00
draft: true
---
# Topic 2: Shared Memory I

> Goal: \
> Explain and motivate how locks, monitors and semaphores can be used to address the challenges caused by concurrent access to shared memory. 

<br><br>
## Motivation for concurrency

### a. Problem in concurrent program 
**Readers and Writers Problem**
* Consider a shared data structure (e.g., an array, list set, …) where threads may read and write. 
* Many threads can read from the structure as long as no thread is writing.
* At most one thread can write at the same time
<br><u>Can we solve this problem using locks?</u>
```java
// The output is any value in the interval (2,20_000) 
long counter = 0;
final long PEOPLE = 10_000;

Turnstile turnstile1 = new Turnstile();
Turnstile turnstile2 = new Turnstile();
turnstile1.start();turnstile2.start();
turnstile2.join();turnstile2.join();
System.out.println(counter+" people entered"); 

public class Turnstile extends Thread {
    public void run() {
        for (int i = 0; i < PEOPLE; i++) {
            counter++;
        }
    }
}

/*
    int temp = counter; // (1)
    counter = temp + 1; // (2)
    
    T1(1) T2(1)                                 Counter=0, T1(temp)=T2(temp)=0
    T1(2) ....... (n-2) * [T1(1) T1(2)]         Counter=X, T2(temp)=0
    T2(2) T1(1)                                 Counter=1, T1(temp)=1
    (n-1) * [T2(1) T2(2)]......T2 finsh         Counter=Y, T1(temp)=1
    T1(2)                                       Counter=2
*/
```
To answer this question we need to understand