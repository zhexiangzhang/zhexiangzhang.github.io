---
title: "[PCPP Note] Topic_5 Performance Measurements"
date: 2023-12-22T19:15:20+01:00
draft: true
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
                dummy += multiply(i);
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