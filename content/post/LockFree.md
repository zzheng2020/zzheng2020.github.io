---
author: ["Ziheng Zhang"]
title: "Lock Free Queue And Time Wheel Mechanism"
date: "2024-09-14"
description: "A C++ Implementation"
summary: "A C++ Implementation"
tags: ["Kubernetes", "Docker", "Golang"]
categories: ["Tech"]
series: ["Master Thesis"]
ShowToc: true
TocOpen: false
---

## Lock-free Queue

In multithreaded programming, queues are frequently used, especially in producer-consumer models. However, using locks (e.g., `std::mutex`) to manage shared queues can degrade performance. In high-concurrency scenarios, lock contention leads to frequent context switching, reducing efficiency. A lock-free queue provides a solution by using atomic operations to ensure thread safety without locks.

Here I am going to show how to implement a thread-safe and lock-free queue using `std::atomic` and a ring buffer in C++ to improve performance (when crital section is relatively small).

### Data Structure for Lock-free Queue

`LockFreeQueue` uses `std::atomic` to manage producer and consumer states, while using a ring buffer for data storage to avoid frequent memory allocations. To enable circular buffer, it uses modulo, ensuring pointers wrap around without exceeding boundaries, thereby eliminating the overhead of dynamic resizing.

```c++
private:
    std::atomic<int> producerHead_;
    std::atomic<int> producerTail_;
    std::atomic<int> consumerHead_;
    std::vector<T> data_{SIZE};

    inline int getIndex(int t) const {
        return t % SIZE;
    }
```

#### `Push` Operation

The main logic of the `push` operation is to update `producerHead_` through the `fetch_add` atomic operation and ensure that the space that the consumer has not read can be written to the data.

```c++
std::optional<int> push(const T& elem) {
    int p_head = producerHead_.fetch_add(1, std::memory_order_seq_cst);
    int p_next = p_head + 1;
    if (consumerHead_.load(std::memory_order_seq_cst) == p_next) {
        // The queue is full.
        return std::nullopt;
    }
    data_[getIndex(p_head)] = std::move(elem);

    // Wait other threads to finish the push operation.
    int wait_cycles = 0;
    while (producerTail_.load(std::memory_order_seq_cst) != p_head) {
        if (wait_cycles % 1000 == 0) {
            std::this_thread::yield();
        }
        wait_cycles++;
    }
    producerTail_.store(p_next, std::memory_order_seq_cst);
    return getIndex(p_head);
}
```

The queue is full by checking if `consumerHead_` is equal to `p_next`. If the consumer's read pointer catches up with the producer's write pointer, the queue is full and no more data can be written.

#### `Pop` Operation

When dequeuing, the consumer reads the data from the location pointed to by `consumerHead_` and advances the pointer.

```c++
bool pop(T& elem) {
    int c_head = consumerHead_.load(std::memory_order_seq_cst);
    int c_next = c_head + 1;
    if (producerTail_.load(std::memory_order_seq_cst) == c_head) {
        return false; // The queue is empty.
    }
    elem = std::move(data_[getIndex(c_head)]);
    consumerHead_.store(c_next, std::memory_order_seq_cst);
    return true;
}
```

## Time Wheel Algorithm

The time wheel is a ring-like structure that divides time into multiple slots (`MAX_TIMEOUT_VALUE` indicates the number of slots), each of which can hold a timer. The time wheel "turns" forward one space after each time period. Whenever the time wheel pointer points to a slot, it checks whether there is an expired timer in the slot. If there is an expired timer, the corresponding callback function is executed.

### Data Structure for Time Wheel

```c++
private:
    std::atomic<uint32_t> now_;
    Callback callback_;
    std::vector<LockFreeQueue<MAX_TIMEOUT_VALUE, uint32_t>> timers_;
```

* `now_`: Current time, which is accessed safely in a multi-threaded environment using the atomic type `std::atomic`. Each time tick is called, `now_` is incremented to advance the time wheel.

* `callback_`: This is the callback function that needs to be called when the timer times out.

* `timers`: Each element is a queue used to store the timer ID.

#### `Set` Function

The timer is set through the `set` method. The timer is assigned to a slot according to its timeout period and stored in the lock-free queue `timers_[]`.

```c++
void set(uint32_t id, uint64_t timeout) {
    assert(timeout > 0);
    assert(timeout < MAX_TIMEOUT_VALUE);

    uint32_t targetTime = (now_ + timeout) % MAX_TIMEOUT_VALUE;
    timers_[targetTime].push(id);
}
```

#### `Tick` Function

The `tick` method is used to advance the time wheel. When the time wheel advances to a certain slot, it checks the timer task in the slot and triggers the callback function.

```c++
void tick(uint32_t milliseconds = 1) {
    for (uint32_t ms = 0; ms < milliseconds; ++ms) {
        now_ = (now_ + 1) % MAX_TIMEOUT_VALUE;
        uint32_t id;
        while (timers_[now_].pop(id)) {
            callback_(id);
        }
    }
}
```