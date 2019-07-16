# LeetCode Concurrency


## [Print in Order](https://leetcode.com/problems/print-in-order/)
Suppose we have a class:

```java
public class Foo {
  public void first() { print("first"); }
  public void second() { print("second"); }
  public void third() { print("third"); }
}

```

The same instance of Foo will be passed to three different threads. Thread A will call ```first()```, 
thread B will call ```second()```, and thread C will call ```third()```. 
Design a mechanism and modify the program to ensure that second() is executed 
after ```first()```, and ```third()``` is executed after ```second()```.

```
Input: [1,2,3]
Output: "firstsecondthird"
Explanation: There are three threads being fired asynchronously. 
The input [1,2,3] means thread A calls first(), thread B calls second(), and thread C calls third(). 
"firstsecondthird" is the correct output.
```

<H3>Solution</H3>

```java

class Foo {

    private CountDownLatch second = new CountDownLatch(1);
    private CountDownLatch third = new CountDownLatch(1);
    
    public void first(Runnable printFirst) throws InterruptedException {
        printFirst.run();
        second.countDown();
    }

    public void second(Runnable printSecond) throws InterruptedException {
        second.await();
        printSecond.run();
        third.countDown();
    }

    public void third(Runnable printThird) throws InterruptedException {
        third.await();
        printThird.run();
    }
}
```

&nbsp;
&nbsp;
## [Print FooBar Alternately](https://leetcode.com/problems/print-foobar-alternately/)
Suppose you are given the following code:

```java

class FooBar {
  public void foo() {
    for (int i = 0; i < n; i++) {
      print("foo");
    }
  }

  public void bar() {
    for (int i = 0; i < n; i++) {
      print("bar");
    }
  }
}

```

The same instance of FooBar will be passed to two different threads. Thread A will call ```foo()``` while 
thread B will call ```bar()```. Modify the given program to output "foobar" n times.

```
Input: n = 1
Output: "foobar"
Explanation: There are two threads being fired asynchronously. One of them calls foo(), while the other calls bar(). "foobar" is being output 1 time.
```

<H3>Solution</H3>

```java

class FooBar {
    private int n;

    public FooBar(int n) {
        this.n = n;
    }
    
    private volatile boolean mutex = true;
    private Lock lock = new ReentrantLock();
    private Condition foo = lock.newCondition();
    private Condition bar = lock.newCondition();

    public void foo(Runnable printFoo) throws InterruptedException {

        for (int i = 0; i < n; i++) {
            lock.lock();
            try {
                while (!mutex) {
                    foo.await();
                }
                bar.signal();
                mutex = false;
                printFoo.run();
            } finally {
                lock.unlock();
            }
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            lock.lock();
            try {
                while (mutex) {
                    bar.await();
                }
                foo.signal();
                mutex = true;
                printBar.run();
            } finally {
                lock.unlock();
            }
        }
    }  
}
```
&nbsp;
&nbsp;
## [Print Zero Even Odd](https://leetcode.com/problems/print-zero-even-odd/)
Suppose you are given the following code:

```
class ZeroEvenOdd {
  public ZeroEvenOdd(int n) { ... }      // constructor
  public void zero(printNumber) { ... }  // only output 0's
  public void even(printNumber) { ... }  // only output even numbers
  public void odd(printNumber) { ... }   // only output odd numbers
}
```

The same instance of ZeroEvenOdd will be passed to three different threads:

Thread A will call zero() which should only output 0's.
Thread B will call even() which should only ouput even numbers.
Thread C will call odd() which should only output odd numbers.
Each of the thread is given a printNumber method to output an integer. 
Modify the given program to output the series 010203040506... where the length of the series must be 2n.

```
Input: n = 2
Output: "0102"
Explanation: There are three threads being fired asynchronously. One of them calls zero(), the other calls even(), and the last one calls odd(). "0102" is the correct output.
```

<H3>Solution: wait + notify</H3>

```java

class ZeroEvenOdd {

    private int n;

    private final int ZERO = 0;
    private final int EVEN = 1;
    private final int ODD = 2;

    private volatile int state = ZERO;

    public ZeroEvenOdd(int n) {
        this.n = n;
    }

    public synchronized void zero(IntConsumer printNumber) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            while (state != ZERO) wait();

            printNumber.accept(0);
            if (i % 2 == 0) {
                state = ODD;
            } else {
                state = EVEN;
            }
            notifyAll();
        }
    }

    public synchronized void even(IntConsumer printNumber) throws InterruptedException {
        for (int i = 2; i <= n; i += 2) {
            while (state != EVEN) wait();
            printNumber.accept(i);
            state = ZERO;
            notifyAll();
        }
    }

    public synchronized void odd(IntConsumer printNumber) throws InterruptedException {
        for (int i = 1; i <= n; i += 2) {
            while (state != ODD) wait();

            printNumber.accept(i);
            state = ZERO;
            notifyAll();
        }
    }
}
```

<H3>Solution: lock + condition</H3>

```java

public class ZeroEvenOdd {
    private int n;
	
    private final int ZERO = 0;
    private final int EVEN = 1;
    private final int ODD = 2;

    private volatile int state = ZERO;
    
    private Lock lock = new ReentrantLock();
    private Condition zero = lock.newCondition();
    private Condition even = lock.newCondition();
    private Condition odd = lock.newCondition();

    public ZeroEvenOdd(int n) {
        this.n = n;
    }

    public void zero(IntConsumer printNumber) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            lock.lock();
            try {
                if (state != ZERO) {
                    zero.await();
                }
                printNumber.accept(0);
                if (i % 2 == 0) {
                    state = ODD;
                    odd.signal();
                } else {
                    state = EVEN;
                    even.signal();
                }
            } finally {
                lock.unlock();
            }
        }
    }
	
    public void even(IntConsumer printNumber) throws InterruptedException {
        for (int i = 2; i <= n; i += 2) {
            lock.lock();
            try {
                if (state != EVEN) {
                    even.await();
                }
                state = ZERO;
                printNumber.accept(i);
                zero.signal();
            } finally {
                lock.unlock();
            }
        }
    }

    public void odd(IntConsumer printNumber) throws InterruptedException {
        for (int i = 1; i <= n; i += 2) {
            lock.lock();
            try {
                if (state != ODD) {
                    odd.await();
                }
                state = ZERO;
                printNumber.accept(i);
                zero.signal();
            } finally {
                lock.unlock();
            }
        }
    }
}
```
