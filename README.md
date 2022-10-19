# Handling Shared Data Part 2

## Learning Goals

- Use the `synchronized` keyword to control access to shared data.
- Explain how synchronizing affects visibility.

## Introduction

We can use the `synchronized` keyword in Java for managing access to a data resource
shared by multiple threads. It can be used in two ways:

- synchronized method.
- synchronized blocks or statements.

## Synchronization

- A **critical section** is a part of a program where shared memory is accessed.

- A **race condition** occurs when a critical section is concurrently
executed by two or more threads.

We can avoid race conditions by synchronizing thread access to the shared data.
The `synchronized` keyword is used to control access to a code block or method.
Only one thread can execute the synchronized code at a time, the other
threads must wait until the first thread finishes executing the synchronized code.

## Static Synchronized Methods

A monitor is a synchronization construct 
that allows a single thread to run the code, while forcing other threads to wait.
When we synchronize a static method using the `syncrhonized` keyword, the class
is used as the monitor.

Let's look at an example.  The `printThread()` method below is not synchronized,
which means the two threads that call the method can take turns
executing the individual statements in the method.

```java
class Example {
    public static void printThread() {
        String curThreadName = Thread.currentThread().getName();
        System.out.println(curThreadName + " entering printThread method");
        System.out.println(curThreadName + " exiting printThread method");
    }

    public static void main(String[] args) {
        new Thread(Example::printThread).start();
        new Thread(Example::printThread).start();
    }
}
```

We can clearly see in the output below that thread execution switches
even when one thread has not completed executing all
the code inside the method.

```text
Thread-1 entering printThread method
Thread-0 entering printThread method
Thread-1 exiting printThread method
Thread-0 exiting printThread method
```

Now let’s add the `synchronized` keyword to the  method signature
and see what happens.  The `Example` class serves as the monitor
since the method is static.

```java
class Example {
    public static synchronized void printThread() {
        String curThreadName = Thread.currentThread().getName();
        System.out.println(curThreadName + " entering printThread method");
        System.out.println(curThreadName + " exiting printThread method");
    }

    public static void main(String[] args) {
        new Thread(Example::printThread).start();
        new Thread(Example::printThread).start();
    }
}
```

This time the first thread completely executes the statements in the
`printThread` method before the second thread can run the method. It is
impossible for multiple threads to concurrently run the code inside
the method.

```text
Thread-0 entering printThread method
Thread-0 exiting printThread method
Thread-1 entering printThread method
Thread-1 exiting printThread method
```

## Instance Synchronized Method

Let's update `printThread()` to be an instance method rather than static. 

Instance methods are synchronized on the instance object. The monitor is the
current object (`this`) that the method was invoked upon.
Each object has a separate monitor for synchronizing.

A single thread can execute the instance method of a particular object.
Multiple threads can run the instance method on different objects, i.e., one
thread can be allocated per object.  However, multiple threads are not allowed
to run the instance method on the same object.

```java
class Example {
    private String name;

    public Example(String name) {
        this.name = name;
    }

    public synchronized void printThread() {
        String curThreadName = Thread.currentThread().getName();
        System.out.println(curThreadName + " entering printThread method of " + name);
        System.out.println(curThreadName + " exiting printThread method of " + name);
    }

    public static void main(String[] args) {
        Example e1 = new Example("Example-Instance-1");
        Example e2 = new Example("Example-Instance-2");

        new Thread(e1::printThread).start(); // Thread-0
        new Thread(e1::printThread).start(); // Thread-1
        new Thread(e2::printThread).start(); // Thread-2
    }
}
```

The output below shows that Thread-0 and Thread-2 concurrently execute the
method because they’re accessing different instances `e1` and `e2`.
However, Thread-1 has to wait for Thread-0 to finish
executing the method before it can start as they’re both working
on the same instance `e1`.


```text
Thread-0 entering printThread method of Example-Instance-1
Thread-2 entering printThread method of Example-Instance-2
Thread-0 exiting printThread method of Example-Instance-1
Thread-2 exiting printThread method of Example-Instance-2
Thread-1 entering printThread method of Example-Instance-1
Thread-1 exiting printThread method of Example-Instance-1
```


Let's look at another example involving the `Counter` class:

```java
public class Counter {
    private int count;

    public int getValue() {
        return count;
    }

    public void increment() {
        count++;
    }
}
```

The program creates two threads to call `increment()` 10 times each:

```java
import java.util.stream.IntStream;

public  class Example {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();

        //each thread calls counter.increment() 10 times
        Thread t1 = new Thread( () -> IntStream.range(0,10).forEach( (i) -> counter.increment()) );
        Thread t2 = new Thread( () -> IntStream.range(0,10).forEach( (i) -> counter.increment()) );

        t1.start();
        t2.start();

        //wait until t1 and t2 finish running
        t1.join();
        t2.join();
        System.out.println("counter value = " + counter.getValue());  //should be 2000
    }
}
```

We saw in the last lesson that `count++` is not an atomic operation as it
consists of 3 steps (read the value of count, add 1 to the value,
write the new value to count), thus  concurrent execution can result in a
final value other than `20`:

```text
counter value = 18
```

We can synchronize the `increment()` method to ensure only one thread
executes the method at a time for a particular `Counter` object:

```java
public synchronized void increment() {
    count++;
}
```

This results in the correct value printed to output:

```text
counter value = 20
```

## Synchronized Blocks

We can synchronize parts of a method by using synchronized blocks.

Let's add a print statement to the `increment()` method.
We don't care if multiple threads execute the print statement at the same time
since it does not involve shared data. However, we do need to
control access to the `count++` statement.

A synchronized block needs to be passed a specific object to use as
the monitor. In the example below, we pass the variable `this` to use the current
`Counter` object.

```java
public class Counter {
    private int count;

    public int getValue() {
        return count;
    }

    public void increment() {
        System.out.println("increment called");
        synchronized (this) {
            count++;
        }
    }
}
```

As we saw with synchronized instance methods,
the synchronized block could execute concurrently in threads
that call the method on a different object, but
concurrent execution is prevented when called on the same object.

If you need to synchronize based on the class instead of the method, use the
`ClassName.class` instead of the `this` keyword with the `synchronized`
statement.


```java
class Example {
    public static void staticMethod() {
        // unsynchronized code
        synchronized (Example.class) {
            // synchronized code
        }
    }
}
```

Synchronizing on the class prevents the block from being executed by multiple
threads at the same time, regardless of how many class instances exist.

## Visibility with Synchronization

The changes performed by a thread are guaranteed to be visible to other threads
that are synchronized on the same monitor. If a thread has modified shared data
inside a synchronized method or block and released the monitor, other threads
can see all changes after acquiring the same monitor.

## Conclusion

We have learned how to use the `synchronized` keyword to make sure that certain
parts of our code can only be executed by one thread at a time. This does
reduce concurrency and the responsiveness of the program so try to synchronize
only the critical sections of code instead of synchronizing whole methods.
