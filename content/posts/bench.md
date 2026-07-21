+++
date = '2026-07-21T13:57:39+10:00'
draft = true
title = 'Bench'
+++

I have been binging performance-related content lately, and the phrase "measure first" comes out a lot. A fond memory of mine of "measuring" is taking a video of an iPhone timer next to our app to prove that the new build is many seconds slower than the previous one. Good old days as a manual QA tester. Now, I am currently trying to write a Single Producer Single Consumer queue that works in the nanosecond range, and hence I started reading about micro-benchmarking, which is what we will go through.

The easiest way to measure how fast a function runs is basically the same as a stopwatch. Measure the start, run the function, and measure the end. Substract the difference, and we have the result.

```java
    public void measure() {
        long start = System.nanoTime();
        functionWeWantToMeasure();
        long end = System.nanoTime();
        long duration = end - start;
        System.out.println("Elapsed nanoseconds: " + duration);
    }
```

For simple checks between two functions, maybe this would suffice as an intial gauge. Why then, do we see tools like Java Microbenchmark Harness (JMH) being used? It comes down to how reliable the above test is. In the above, we only run the code once, which could prevent the JIT compiler to optimize the code. We are hence quite possibly measuring the interpreted code, not how it would actually perform in a long running process. To address that, we could  write a loop, accumulate the average, and output the result. We could do that, but all of this is already done in JMH. We simply use the annotations given instead of handrolling our own benchmark every time.

How can we setup the JMH then? Here are the steps to run a JMH benchmark, assuming that we are using Gradle (Kotlin DSL) and Java, which is the stack that I like to use.

## Setup

1. Add the JMH plugin to `build.gradle.kts`.

    Refer to https://github.com/melix/jmh-gradle-plugin

    ```
    plugins {
        id("me.champeau.jmh") version "0.7.3"
    }
    ```

    Run `./gradlew build` to download the plugin.


1. Create a folder src/jmh/java

    This is the default location where the tool expects to find your benchmarks. Put your benchmarks here.

2. Write the function that you want to test 

    I was curious on the effect of autoboxing on performance, and hence I wrote these two functions to try and capture that.

```java
public class Sum {
    public static int withPrimitives(int start, int end) {
        int result = 0;
        for (int i = start; i <= end; i++) {
            result += i;
        }
        return result;
    }

    public static Integer withAutoBoxing(int start, int end) {
        Integer result = 0;
        for (int i = start; i <= end; i++) {
            result += i;
        }
        return result;
    }
}

```
1. Write your benchmark
    Create a `SumBenchmark.java` in `/src/jmh/org/example/`. 
    The benchmark below sets up the range of numbers that we are summing up, and defining the two functions to test.

```java
public class SumBenchmark {
    private final static int START_NUM = 1;
    private final static int END_NUM = 10_000;

    @Benchmark
    public void sumWithPrimitive(Blackhole bh) {
        bh.consume(Sum.withPrimitives(START_NUM, END_NUM));
    }
    
    @Benchmark
    public void sumWithAutoBoxing(Blackhole bh) {
        bh.consume(Sum.withAutoBoxing(START_NUM, END_NUM));
    }
}
```

By default, the JMH will run a throughput test, which tests how many times we can complete the function. Our hypothesis is that the AutoBoxing will have a lower throughput compared to the primitive function. 

`bh.consume` is a function that uses the result from our function to prevent the compiler from removing our code. This happens when the compiler sees that values are not being used, and we would essentially be measuring nothing.

Let's test that now with `./gradlew jmh`. Note, if you run above, it will take a fairly long time because of JMH's default settings. You can see the defaults in the terminal after you run JMH:

```
# Warmup: 5 iterations, 10 s each
# Measurement: 5 iterations, 10 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Benchmark: org.example.SumBenchmark.sumWithAutoBoxing
```

If you are impatient and would like to just see a sample result, feel free to modify the configs. You could for example reduce the measurement and warmup time as follows:

```java
@Measurement(iterations = 5, time = 1)
@Warmup(iterations = 5, time = 1)
public class SumBenchmark {
    // the rest of the code.
```

## Results

We can see from results below that true enough, summing using a primitive is ~3.5x times faster than with autoboxing.

```
Benchmark                        Mode  Cnt       Score      Error  Units
SumBenchmark.sumWithAutoBoxing  thrpt   25   81416.429 ± 2077.866  ops/s
SumBenchmark.sumWithPrimitive   thrpt   25  297518.357 ± 2719.078  ops/s
```
Machine used is Macbook Air M1 2020.

This is expected due to the creation of Integer objects on the heap is an additional operation. On top of that, as heap objects continue to accumulate, the garbage collector might run to move objects from the Eden space.

And there you go, not too hard to measure!

Well, just to start anyway. There are so many things that would affect our measurements (and what we are actually measuring) such as:
- Dead Code Elimination
- Constant Folding
- Cache locality

As a summary, we managed to setup JMH and run a simple benchmark. Now, we are free to measure anything we want. 

Funny things:

I added a test where I did not use a black hole:
```java
@Benchmark
public void sumWithPrimitiveNoBh(Blackhole bh) {
    int result = Sum.withPrimitives(START_NUM, END_NUM);
}
```
```
SumBenchmark.sumWithPrimitive                                    thrpt      25      297820.138 ±      4679.348  ops/s
SumBenchmark.sumWithPrimitiveNoBh                                thrpt      25  1693983257.141 ± 107050001.440  ops/s
```
`sumWithPrimitiveNoBh` seems to be working _amazingly_ well. Theory says that we're measuring nothing, but I'll need to grab a disassembler to really make sure. That's for another time :)
