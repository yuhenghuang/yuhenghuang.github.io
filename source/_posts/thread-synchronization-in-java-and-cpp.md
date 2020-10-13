---
title: Thread Safety in cpp and Java
top: false
cover: false
toc: true
mathjax: false
date: 2020-10-11 09:31:03
password:
summary:
tags:
  - Thread
categories:
  cpp
---


The post exhibits two examples of thread synchronization (or communication) in *cpp* and *java*, with comments on the basic concepts.


# Thread-safe Queue

First thing comes the thread lock `std::mutex`. This lock cannot be copy or move to other objects thus it ensures only one thread shall own it.

`std::lock_guard` and `std::unique_lock` are wrappers of `std::mutex` which provide more functionalities of thread lock.


`std::condition_variable` controls whether a lock should be kept or released by the variants of `wait()` and `notify()` methods.


The templated ThreadQueue has a `std::queue<T>`, `std::mutex`, `std::condition_variable` and optionally functor `Predicate`, end condition `T`.


The basic workflow of is in a multithread environment,

thread that calls `push` or `pop` first will take the `std::mutex` by a wrapper and prevent other threads from doing that. At this point, all other thread will be waiting at line `std::lock_guard<std::mutex>` or  `std::unique_lock<std::mutex>`.

If the awaken thread is in `push`, it will execute to the end of the scope.

If the awaken thread is in `pop`, the story will be different. After it took the thread lock, the coming `condition_variable` won't let it pass with the lock unless

* Received notification from other threads (by `notify()` or `notify_all()`)
* (Optional) the condition is satisfied

In the example we check whether the `std::queue` is empty, but there can also be other types of condition like `timeout`.


```cpp

template<typename T, class Predicate=std::equal_to<T>>
class ThreadQueue {
  private:
    std::queue<T> q;
    std::mutex m; // only one thread can either uses push or pop.
    std::condition_variable cv;
    T end_condition;
    Predicate pred;

  public:
    ThreadQueue(const T& ec): end_condition(ec) {}

    void push(T&& elem) {
      std::lock_guard<std::mutex> lock(m);
      
      q.push(elem);

      cv.notify_one();
    }

    bool pop(T& elem) {
      // own mutex
      std::unique_lock<std::mutex> lock(m);
      /*
        1. mutex released and execution suspended before awaking (being awakened)
        2. Being awakened by notify_one() ( or notify_all() )
        3. mutex reacquired, check the condition
        4. If true -> move to next line, else -> go back to 1
      */
      cv.wait(lock, [this](){ return !q.empty(); });

      // if is not necessary?
      if (!q.empty()) {
        elem = std::move(q.front());
        q.pop();
      }

      return pred(elem, end_condition) ? false : true;
    }

};

```

The `pop` method returning `bool` instead of `T` has to merits.

* Avoid potential unnecessary copies (even compiler will optimize some of those implicitly)
* Inform the caller of the method

Like in the implementation, the `pop` method will return `false` if the popped element is the end_condition we defined at the time of construction. The program can be designed to be more flexible.


We also need to write the specialization of `std::equal_to<T>` if `T::operator==` is not defined.
The example using `std::string` looks like

```cpp

namespace std {
  template<>
  struct equal_to<string> {
    bool operator()(const string& lhs, const string& rhs) const {
      return lhs.compare(rhs) == 0;
    }
  };
}

```



### Print Odd-Even Number in Turn

In *Java*, the thread lock is already defined in the `Object` class, as well as `notify()` and `wait()` method.

The scope that is expected to be thread safe is indicated by *Java* keyword `synchronized(Object obj)`.

Following code prints odd and even numbers by two threads in turn.


```java
class PrintThread implements Runnable {
  int rem;
  Number num;

  PrintThread(Number n, boolean is_even) {
    num = n;
    rem = is_even ? 0 : 1;
  }

  @Override
  public void run() {
    while (num.i<num.max) 
      num.print(rem);
  }
}

class Number {
  public static void main(String[] args) {
    Number obj = new Number(200);

    Thread t1 = new Thread(new PrintThread(obj, true));
    t1.setName("t1");

    Thread t2 = new Thread(new PrintThread(obj, false));
    t2.setName("t2");

    t1.start();
    t2.start();
  }

  int i, max;

  Number(int m) {
    i = 1;
    max = m;
  }

  public void print(int rem) {
    synchronized (this) {
      if (i % 2 != rem) {
        try {
          wait();
        } 
        catch (InterruptedException e) { }
      }

      System.out.println(Thread.currentThread().getName() + " -> " + i++);
      notify();
    }
  }
}
```
