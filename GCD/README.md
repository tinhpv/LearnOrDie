## Grand Central Dispatch (GCD)
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
### Serial queue
- a serial queue is created, and a task is dispatched to that queue, system creates one thread for it and it is independent with other serial queues. 
- For example, 2 serial queues are created, started at the same time, will start running simultaneously.

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
/// Both queues here are serial, but the results are jumbled up because they execute concurrently in relation to each other
/// Their QoS level determines who will generally finish first (order not guaranteed)
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
- want the first queue to be done first then the next queue execute their task? → use `sync`, but it will block the caller.
- another way? → use one single queue for 2 tasks.
```swift
serial1.async {
    for _ in 0..<5 { print("🔵") }
}

serial1.async {
    for _ in 0..<5 { print("🔴") }
}

/// CONSOLE
🔵
🔵
🔵
🔵
🔵
🔴
🔴
🔴
🔴
🔴
```

----
### sync/async vs. concurrent/serial
sync/async and concurrent/serial are two **SEPARATE** concepts.
- Synchronous vs. asynchronous is about when THE CALLER (the queue) can continue. 
	- `sync` **block** the *current queue* until that task completes. Once the current task is completes/returns, another task can be dispatched to the queue.
	- `async` execute **asynchronously** with respect to the *current queue*. Another task can be dispatched to the queue (BUT that task can be run or not depends on the queue is serial or concurrent)

> ► GCD optimizes performance by executing that task on the current thread (the caller.) 
> ► There is one exception however, which is when you submit a sync task to the main queue — doing so will always run the task on the main thread and not the caller. 

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
- the first can’t finish because it’s waiting for the second to finish, but the second can’t finish because it’s waiting for the first to finish.

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

*Another example:*
```swift
let concurrentQueue = DispatchQueue(label: "concurrentQueue", attributes: .concurrent)

concurrentQueue.sync { // ← block the caller
    for _ in 0..<5 { print("🔵") }
}

concurrentQueue.async { // ← try to dispatch a task to the queue while it executing the sync statement
    for _ in 0..<5 { print("🔴") }
}

/// CONSOLE
🔵
🔵
🔵
🔵
🔵
🔴
🔴
🔴
🔴
🔴
```
---
### Priority Inversion
- a high priority task is prevented from running by a lower priority task
- often occurs when a high QoS queue shares a resources with a low QoS queue, and the low QoS queue gets a lock on that resource

E.g:
- scenario: submit a task to low-QoS serial queue → submit another task that have high priority to that queue.
- problem: high-priority task waits for low-priority task to be finished
- resolve: temporarily increase the QoS of the queue of the low-priority task

```swift
let starterQueue = DispatchQueue(label: "starter.queue", qos: .userInteractive)
let utilityQueue = DispatchQueue(label: "utility.queue", qos: .utility)
let backgroundQueue = DispatchQueue(label: "background.queue", qos: .background)

starterQueue.async {
    backgroundQueue.async { 
        output(color: .white)
    }

    utilityQueue.async {
        output(color: .blue)
    }

    backgroundQueue.sync {} // this one make priority inversion
}

/// CONSOLE
/// blue circles were printed last
⚪️
🔵
⚪️
⚪️
⚪️
🔵
⚪️
🔵
🔵
🔵
```
- the last `sync` statement run on the caller thread (starterQueue with top priority), it blocks the queue now (1)
- but another task that has been submited to the same queue but asynchronously (2)
- (1) have to wait for (2) (cuz starter is a serial queue) while (1) has higher priority than (2)

→ GCD increases the QoS of the background queue to temporarily match the high QoS task (userInteractive)
→ background queue is now having high priority than the other (utility)

---
### Thread explosion 
- try to submit tasks to a concurrent queue that is currently blocked (e.g. with a semaphore, sync, or some other way.) 
- These tasks will run, but the system will likely end up spinning up new threads to accommodate these new tasks.
- Apple suggests starting with a serial queue per subsystem in your app, as each serial queue can only use one thread at a time.

> Serial queues are concurrent in relation to other queues, so you still get a performance benefit when you offload your work to a queue, even if it isn't concurrent.

---
### Race Condition, Solved By Dispatch Barrier
- Readers-writers problem
e.g: multiple threads access/modify the same resource (as an array, a file)

One of solutions: using an isolation queue
use `barrier flag` - whether it’s allowed to run concurrently with any other dispatched blocks to that queue
- submitting a task without barrier flag, the queue works as a normal concurrent queue
- when the barrier is executing, it is working as a serial queue.
- blocks any further tasks from executing until the barrier task is completed.
- use `barrier` flag on a serial queue is redundant 🤷‍♂️

> A block submitted with this flag will act as a barrier: 
> - All other blocks that were submitted BEFORE the barrier will finish and only then the barrier block will execute.
> - All blocks submitted AFTER the barrier will not start until the barrier has finished.

⛔️ **Barriers do not work on global queues; they only affect private concurrent queues that you created** ⛔️
Why?
> → Global queues are shared. You’re not the only one possibly availing yourself of these queues. Other subsystems in your app might be using them. The OS might, too. And barriers are a blocking operation, which could have serious impact if they started blocking unrelated systems. It seems exceeding prudent to me that GCD prevents one a bit of code in one subsystem from blocking all the other completely unrelated subsystems

```swift
let isolation = DispatchQueue(label: "queue.isolation", attributes: .concurrent)
private var _array = [1,2,3,4,5]
var threadSafeArray: [Int] {
       get {
            return isolation.sync { // ← don't let another task asynchronously cut
                _array
            }
        }
        set {
            isolation.async(flags: .barrier) {
                self._array = newValue
            }
        }
}
```
could use a serial queue without a barrier to solve the race condition
but then we would lose the advantage of having concurrent read access to the array.

---

### Summary - pros and cons
- to make the app highly responsive.
- on OSX and iOS, the main loop, called RunLoop on the main thread, should not be blocked - only the main thread can update UI. When it is blocked, UI is not updated and the same still image is displayed for quite a long time → frozen 🥶

BUT...
- multiple threads compete to update the same resource, it causes inconsistent data (called a **race condition**)
- multiple threads await an event at the same time (**deadlock**)
- thread explosion, when too many threads are used, the application memory becomes short


Learned from:
- https://www.avanderlee.com/swift/concurrent-serial-dispatchqueue/
- https://www.donnywals.com/understanding-how-dispatchqueue-sync-can-cause-deadlocks/
- https://medium.com/@almalehdev/concurrency-visualized-part-2-serial-vs-concurrent-fd04e32c20a9
- https://medium.com/@almalehdev/concurrency-visualized-part-3-pitfalls-and-conclusion-2b893e04b97d
- https://developer.apple.com/forums/thread/106319
- https://stackoverflow.com/questions/71233769/why-concurrent-queue-with-sync-act-like-serial-queue
- https://stackoverflow.com/a/58238703
