# Handling Shared Data Part 2

## Learning Goals

- Use the `synchronized` keyword in Java.
- Explain how synchronizing affects visibility.

## Introduction

We can use the `synchronized` keyword in Java for managing access to a resource
by multiple threads. It can be used in two ways:

- synchronized method.
- synchronized blocks or statements.

Both of these can be used for either static or instance methods.

## Static Synchronized Methods

When we synchronize a static method using the `syncrhonized` keyword the class
is used as the monitor. A single thread is allowed to run the code in the static
method while the other threads have to wait for the first thread to complete its
execution before running the static method.

Here’s an example without the a synchronized method. We can clearly see that
thread execution switches even when one thread hasn’t completed executing all
the code inside the method.

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

```plaintext
Thread-1 entering printThread method
Thread-0 entering printThread method
Thread-1 exiting printThread method
Thread-0 exiting printThread method
```

Now let’s add the `synchronized` keyword and see what happens.

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

```
Thread-0 entering printThread method
Thread-0 exiting printThread method
Thread-1 entering printThread method
Thread-1 exiting printThread method
```

This time the first thread completely executes the statements in the
`printThread` method before the second thread can run the method. It is
impossible for multiple threads to run the code inside the `printThread` method
at the same time.

## Instance Synchronized Method

Instance methods are synchronized on the instance object. The monitor is the
current instance (`this`) that owns the method. Each instance has a separate
monitor for synchronizing.

A single thread can execute the instance method of a particular instance.
Multiple threads can run the instance method on different instances, i.e., one
thread can be allocated per instance.

```java
class Example {
    private final String name;

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

```plaintext
Thread-0 entering printThread method of Example-Instance-1
Thread-2 entering printThread method of Example-Instance-2
Thread-0 exiting printThread method of Example-Instance-1
Thread-2 exiting printThread method of Example-Instance-2
Thread-1 entering printThread method of Example-Instance-1
Thread-1 exiting printThread method of Example-Instance-1
```

Notice that Thread-0 and Thread-2 are running at the same time because they’re
accessing different instances. But Thread-1 has to wait for Thread-0 to finish
running before it can start as they’re both working on the same instance.

## Synchronized Blocks

We can synchronize only parts of a method by using synchronized blocks. The
synchronized blocks need to be passed in a specific object for locking threads.

```java
class Counter implements Runnable {
    private int count = 0;

    public void increment() {
        synchronized (this) {
            count++;
        }
    }

    public int getValue() {
        return count;
    }

    @Override
    public void run() {
        String curThreadName = Thread.currentThread().getName();
        System.out.println(curThreadName + " before increment count = " + count);
        this.increment();
        System.out.println(curThreadName + " after increment count = " + count);
    }
}

class Example {
    public static void main(String[] args) {
        Counter counter = new Counter();
        Thread t1 = new Thread(counter);
        Thread t2 = new Thread(counter);

        t1.start();
        t2.start();
    }
}
```

```plaintext
Thread-1 before increment count = 0
Thread-0 before increment count = 0
Thread-1 after increment count = 1
Thread-0 after increment count = 2
```

The `run` method isn’t synchronized so the threads can switch execution when
they’re in the `run` method. When a thread is running the `increment` method, it
complete the execution of the method before another thread can access the
method. This ensures that the `count` value is always incremented consistently.

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

## Visibility with Synchronization

The changes performed by a thread are guaranteed to be visible to other threads
that are synchronized on the same monitor. If a thread has modified shared data
inside a synchronized method or block and released the monitor, other threads
can see all changes after acquiring the same monitor.

## Conclusion

We have learned how to use the `synchronized` keyword to make sure that certain
parts of our code can only be executed by one thread at the same time. This does
reduce parallelism and the responsiveness of the program so try to synchronize
only small parts of the code instead of synchronizing whole methods.
