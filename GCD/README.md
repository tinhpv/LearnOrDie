### Grand Central Dispatch (GCD)
---
- GCD helps to write multi-threaded code, create threads and schedule tasks on those threads.
- Main building block: **Dispatch Queues**, 3 types of queues:
    - **Main dispatch queue** (serial, pre-defined)
        - drawing app's UI 🎨
        - handle events (e.g. user interaction) 
        - block it for too long, app will freeze 🥶
    - **Global queue** (concurrent, pre-defined)
    - **Private queue** (can be serial/concurrent optionally, serial by default)


*Usage:*
- Making a network call, which takes a long time in background queue
- When it completes, updating UI on main thread.
```swift
URLSession.shared.dataTask(with: url) { data, response, error in
    if let data = data {
        DispatchQueue.main.async { 
            self.label.text = String(data: data, encoding: .utf8)
        }
    }
}
```
----
#### Serial queue
- a serial queue is created, and a task is dispatched to that queue, system creates one thread for it and it is independent with other serial queues. For example, 2 serial queues are created, started at the same time, will start running simultaneously.

```swift
let serial1 = DispatchQueue(label: "tinhpv.serial1")
let serial2 = DispatchQueue(label: "tinhpv.serial2")

serial1.async {
    for _ in 0..<5 { print("🔵") }
}

serial2.async {
    for _ in 0..<5 { print("🔴") }
}

/// CONSOLE
🔵
🔴
🔴
🔴
🔵
🔴
🔵
🔴
🔵
🔵
```
- 
----
#### sync vs async - concurrent vs. serial
sync/async and concurrent/serial are two **SEPARATE** concepts.
- Synchronous vs. asynchronous is about when THE CALLER (the queue) can continue. 
    - `sync` **block** the *current queue* until that task completes. Once the current task is completes/returns, another task can be dispatched to the queue.
    - `async` execute **asynchronously** with respect to the *current queue*. Another task can be dispatched to the queue (BUT that task can be run or not depends on the queue is serial or concurrent)
- Concurrent vs. serial is about when the DISPATCHED TASK of the queue can run.
    - *serial*
        - the task maybe cannot run immediately if the queue is already running some other tasks. 
        - NO more than 1 thread at a time. 
        - Guarantee order
    - *concurrent*
        - multiple threads, system decides number of threads.
        - Do not wait each other, order is not guaranteed.

*Example:*
```swift
let queue = DispatchQueue(label: "queue_label")
queue.sync { // (1) submit a task synchronously into the queue
    queue.sync { // (2)
        // dead lock ☠️
    }
}
```
☠️ why deadlock?
- first task is executing, which submits another task to the queue SYNCHRONOUSLY.
- "SYNCHRONOUSLY" = first task have to DONE so that the next task can be dispatched to the queue.
- so, second task DOES NOT return because it's waiting for the first task to be done.
- also, this is a SERIAL queue, have to execute tasks in order (first task have to be done > second task could starts)
- ... and the first task canNOT be done because of the second task.
- 1st task waits for 2nd task and conversely. >>> DEADLOCK!

Another one:
```swift
let queue = DispatchQueue(label: "queue_label")
queue.async { // ← (1) change this to async
    queue.sync { // (2)
        // Still dead lock ☠️
    }
}
```
☠️ why still deadlock?
- now first task is dispatched ASYNCHONOUS to the queue and the queue executes this task.
- this task is dispatching another task to the queue but now it is SYNCHRONOUS.
- the queue is now is BLOCKED by the `sync` statement until the second completes, but the second task cannot be started because the queue is serial, wait for the first task, but the first task is not returned because the second task (its content) is not returned either!

SOLUTION:
```swift
let queue = DispatchQueue(label: "queue_label")
queue.sync { // ← sync or async is still working
    queue.async { // ← make it async
        print("Inner!")
    }
}
```
- the first task was submitted async or sync is still working.
- in case the first task is sync, it blocks the queue until the content inside returned.
- as per content of the first task, it's dispatching asynchronously another task to the queue, that `async` statement can be returned immediately (that second task is successfully dispatched to the queue, but it is still not yet executed, because this queue is serial, have to wait for the first task to be done)
- so now the first task could be done and it is time for the second task

#### Advantages and Disadvantages of Multithreaded Programming
- to make the app highly responsive.
- on OSX and iOS, the main loop, called RunLoop on the main thread, should not be blocked - only the main thread can update UI. When it is blocked, UI is not updated and the same still image is displayed for quite a long time → frozen 🥶

BUT...
- multiple threads compete to update the same resource, it causes inconsistent data (called a **race condition**)
- multiple threads await an event at the same time (**deadlock**)
- when too many threads are used, the application memory becomes short

Learned from:
- https://www.avanderlee.com/swift/concurrent-serial-dispatchqueue/
- https://www.donnywals.com/understanding-how-dispatchqueue-sync-can-cause-deadlocks/
- https://medium.com/@almalehdev/concurrency-visualized-part-2-serial-vs-concurrent-fd04e32c20a9
- https://developer.apple.com/forums/thread/106319
- https://stackoverflow.com/questions/71233769/why-concurrent-queue-with-sync-act-like-serial-queue
