# Async/Await - AsyncSequence - Actor

```swift
func download(url: URL, completionHandler: @escaping (Data) -> ()) {
    let task = URLSession.shared.dataTask(with: url) { data, _, error in
        if let data = data {
            completionHandler(data)
        }
    }
    task.resume()
}
```
⬇⬇⬇
```swift
func download(url: URL) async throws -> Data {
    let result = try await URLSession.shared.data(from: url)
    return result.0 // the response data
}
```
- async code runs IN ORDER. No inversion as using completion handler
- async code can return value to its caller.
- async code can throw error

**await**
- used to call `async` methods.
- cause our code to *pause* and *wait* until the call to `async` method finishes. BUT, it does NOT block.
- use `await` on background thread, but also legal to put it on main thread.

❓**but, only in an `async` context, how about calling an async method from a normal method?**
e.g. call `download(from:)` in `viewDidLoad()` of `UIViewController`

👇
## Tasks
- This is `a unit of asynchronous work`
- Call `Task.init(operation:)`, it *inherits* the characteristics of its surroundings
```swift
// ✅  RUN ON MAIN THREAD
override func viewDidLoad() {
    super.viewDidLoad()
    
    Task {
    // ✅ STILL RUN ON MAIN THREAD
        do {
            let imgUrl = "https://cdn.pixabay.com/photo/2015/04/23/22/00/tree-736885__480.jpg"
            let data = try await download(url: URL(string: imgUrl)!)
            print(data)
        } catch(let error) {
            print(error.localizedDescription)
        }
    }
    print(imgUrl)
}

func download(url: URL) async throws -> Data {
    let result = try await URLSession.shared.data(from: url)
    return result.0 // the response data
}
	
///  CONSOLE
https://cdn.pixabay.com/photo/2015/04/23/22/00/tree-736885__480.jpg
44304 bytes
```
- Call `Task.detached(operation:)`, it cuts off relationship to the surrounding context, running on its own background thread
```swift
// ✅  RUN ON MAIN THREAD
override func viewDidLoad() {
    super.viewDidLoad()
    
    Task.detached {
        // ❇️ NOW, RUN ON BACKGROUND THREAD
        ...
    }
    print(imgUrl)
}
```
## Wrapping a Completion Handler
⁉️*Want to adopt structured concurrency but keep old-fashioned async code based on completion handler?*
✅ Wrap it with 
- `withUnsafeContinuation(_:)`
- `withUnsafeThrowingContinuation(_:)`
- `withCheckedContinuation(_:)`
- `withCheckedThrowingContinuation(_:)`

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    let imgUrl = "https://cdn.pixabay.com/photo/2015/04/23/22/00/tree-736885__480.jpg"
    Task {
        do {
            let data = try await download(url: URL(string: imgUrl)!)
            print(data)
        } catch(let error) {
            print(error.localizedDescription)
        }
    }
    print(imgUrl)
}

func download(url: URL) async throws -> Data {
    return try await withUnsafeThrowingContinuation { continuation in
        download(url: url) { data in
            continuation.resume(returning: data) // ⚠️ call resume ONLY ONCE
        }
    }
    
//        let result = try await URLSession.shared.data(from: url)
//        return result.0 // the response data
}

// The old-fashion async functino with completion handler
func download(url: URL, completionHandler: @escaping (Data) -> ()) {
    let task = URLSession.shared.dataTask(with: url) { data, _, error in
        if let data = data {
            completionHandler(data)
        }
    }
    task.resume()
}
```
> `UnsafeContinuation` is basically the same kind of object that forms the basis of the `await` keyword itself. 
- when we wait for an async function call using `await`, a continuation object is created and stored behind the scenes 
- when the async function finishes, a method of that continuation object is called, and our code resumes. So what we’re doing in this wrapper method is enacting manually the same sort of wait-and-resume architecture that is performed for us automatically when we say `await`.

**Continuation**?
A mechanism to interface between synchronous code and an asynchronous stream

## Do multiple concurrent tasks with `async let`
Don't do this, it is serial, not concurrent:
```swift
func fetchTwoURLs() async throws -> (Data, Data) {
    let url1 = URL(string:"https://www.apeth.com/pep/manny.jpg")!
    let url2 = URL(string:"https://www.apeth.com/pep/moe.jpg")!
    let data1 = try await self.download(url: url1)
    let data2 = try await self.download(url: url2) /// 😢 this one waits for the above ☝ to be done 
    return (data1, data2)
}
```
Do this instead, using `async let`:
```swift
func fetchTwoURLs() async throws -> (Data, Data) {
    let url1 = URL(string:"https://www.tinhpv.com/pep/manny.jpg")!
    let url2 = URL(string:"https://www.tinh.com/pep/moe.jpg")!
    async let data1 = self.download(url: url1) /// (1)
    async let data2 = self.download(url: url2) /// (2)
    return (try await data1, try await data2) /// try wait (data1, data2)
}
```
- *when we know the number of tasks to do* (if perform an indeterminate number of tasks, use task group)
- Useful when we want to return the results of both `download(url:)` calls as a single result. Both calls need to finish before return statement.
- ⚠️ if the `async` function throws an error, it does NOT interrupt code. For e.g., when (1) fails, throws error, (2) still goes ahead and still be awaited. The thrown error from the subtask doesn't interrupt our code until later, when we use `try`.

another style for writing `async let`:
```swift
func fetchTwoURLs() async throws -> (Data, Data) {
    let url1 = URL(string:"https://www.tinhpv.com/pep/manny.jpg")!
    let url2 = URL(string:"https://www.tinh.com/pep/moe.jpg")!
//    async let data1 = self.download(url: url1) /// (1)
//    async let data2 = self.download(url: url2) /// (2)

        
    async let data1 = {         
        return try await download(url: url1)
    }()
                     
    async let data2 = {
        return try await download(url: url2)
    }()    
    
    return (try await data1, try await data2) /// try wait (data1, data2)
}
```

## Task Groups
use 1 of these 2 methods:
- `withTaskGroup(of:returning:body:)`
- `withThrowingTaskGroup(of:returning:body:)`
e.g.:
```swift
func fetchData(from urls: [URL]) async throws -> [Data] {
    var result: [Data] = []
    
    try await withThrowingTaskGroup(of: Data.self) { group in
        for url in urls {
            group.addTask {
                return try await self.download(url: url)
            }
        }
        
        // loop through the group as an async sequence
        // get the value from each subtasks and append to the list result.
        for try await data in group {
            result.append(data)
        }
    }
    return result
}
```
... But result is not in order as the url array that being passed in because it is concurrent.
so rather than return an array, we return a dictionary.
```swift
func fetchData(from urls: [URL]) async throws -> [URL: Data] {
    var result: [URL: Data] = [:]
    
    try await withThrowingTaskGroup(of: [URL: Data].self) { group in
        for url in urls {
            group.addTask {
                // this return here have to match with `of` type
                return [url: try await self.download(url: url)]
            }
        }
        
        for try await dictionaryData in group {
            // merge dictionaryData to result dictionary
            // if there is a duplicate key, keep the key of result
            result.merge(dictionaryData) { current, _ in current }
        }
    }
    
    return result
}
```
Full code 🐣
```swift
func fetchData(from urls: [URL]) async throws -> [URL: Data] {
    var result: [URL: Data] = [:]
    
    return try await withThrowingTaskGroup(of: [URL: Data].self, returning: [URL: Data].self) { group in
        for url in urls {
            group.addTask {
                // this return here have to match with `of` type
                return [url: try await self.download(url: url)]
            }
        }
        
        for try await dictionaryData in group {
            // merge dictionaryData to result dictionary
            // if there is a duplicate key, keep the key of result
            result.merge(dictionaryData) { current, _ in current }
        }
        
        return result
    }
}
```
- specify `returning`, denoting that this block will return with the type of `returning` ([URL: Data] dictionary as above)

## Asynchronous Sequence
- task group as above is an asynchronous sequence
- an asynchronous sequence is a type that conforms to `AsyncSequence` protocol
- values of an asynchronous sequence are generated ASYNCHRONOUSLY
> An AsyncSequence may have all, some, or none of its values available when you first use it. Instead, you use await to receive values as they become available.
- `for await` loop pauses after each iteration → resumes when the next value of the sequence is generated → performing another iteration → waiting again, until the sequence signals that it has terminated.

### Create an Asynchronous Sequence
#### use the `AsyncStream` initializer (or AsyncThrowingStream)
- as a stream of elements that could potentially result in a thrown error. 
- values deliver over time, and the stream can be closed by a finish event.
- a finish event could either be a success ✅ or a failure ❌ once an error occurs.

> An asynchronous sequence generated from a closure that calls a continuation
> - `AsyncStream` conforms to `AsyncSequence`, providing a convenient way to create an asynchronous sequence without manually implementing an asynchronous iterator. 
> - In particular, an asynchronous stream is well-suited to adapt CALLBACK- or DELEGATION-based APIs to participate with `async`-`await`.

```swift
class TextFieldStreamer: NSObject, UITextFieldDelegate {
    var values: AsyncStream<UITextField>
    var continuation: AsyncStream<UITextField>.Continuation?
    
    override init() {
        var myContinuation: AsyncStream<UITextField>.Continuation?
        self.values = AsyncStream { continuation in
            myContinuation = continuation
        }
        super.init()
        self.continuation = myContinuation
    }
    
    func textFieldDidChangeSelection(_ textField: UITextField) {
        self.continuation?.yield(textField)
    }
}

/// USAGE
textField.delegate = self.textFieldStreamer
Task {
	for await textField in textFieldStreamer.values {
		print(textField.text ?? "")
	}
}
```

Another example from avanderlee.com, function download return an async sequence with value of type `Status`
```swift
extension FileDownloader {
    func download(_ url: URL) -> AsyncThrowingStream<Status, Error> {
        return AsyncThrowingStream { continuation in
            do {
                try self.download(url, progressHandler: { progress in
                    continuation.yield(.downloading(progress))
                }, completion: { result in
                    switch result {
                    case .success(let data):
                        continuation.yield(.finished(data))
                        continuation.finish()
                    case .failure(let error):
                        continuation.finish(throwing: error)
                    }
                })
            } catch {
                continuation.finish(throwing: error)
            }
        }
    }
}
```
☝
If using `AsyncStream` instead of `AsyncThrowingStream`, it accepts 1 parameter only and cannot finish with error as `continuation.finish(throwing: error)`

`func download(_ url: URL) -> AsyncStream<Status> {}`

---
## Actors
- Kind object type, on a par with `enum`, `class`, `struct`
- It is a **reference type** like a `class`
- Actor's code run on *background* by default, it has *isolation domain*, which helps multithreaded code run safely.
### Actor Isolation
1. if an actor instance have *mutable property* (`var`), only that instance can mutate it.
2. Can access actor's **property**, but this access is *asynchronous*, must use `await`
3. Can call actor's **method**, but this access is *asynchronous*, must use `await` (even if it is not an `async` method)

Why `await`❓ To make sure, no other code belonging to the same actor can start, solves the problem of simultaneous access. With an actor, there is no simultaneous access. Access to an actor is serialized. ✅

```swift
actor MyActor {
    
    let id = UUID().uuidString 
    var actorProperty: String
    
    init(actorProperty: String) {
        self.actorProperty = actorProperty
    }
    
    func mutateProperty(_ newValue: String) {
        self.actorProperty = newValue
    }
}

let actor = MyActor(actorProperty: "MyActor")
let id = actor.id // ✅ id is constant value, can directly access
let property = actor.actorProperty // ⛔️🆘 Actor-isolated property, must use `await`
actor.actorProperty = "Vietnam" // ⛔️🆘 it is not allowed, refer (1) above

Task {
    let property = await actor.actorProperty // ✅ look good now
    await actor.mutateProperty("Vietnam") // ✅ it is legal
}
```

## Main Actor
- Isolate code to main actor, it will run on main thread
- How? Use `@MainActor` directive on:
  - *property* (static/class/instance property) 👉 only be accessed on main thread
  - *method* (static/class/instance property) 👉 only be called on main thread
  - *a type (class, struct, enum)* 👉 all of its properties and methods are only accessed only on the main thread.
  - *global variable or function* 
   
```swift
  actor MyActor {
    
    let id = UUID().uuidString
    @MainActor var actorProperty: String
    
    init(actorProperty: String) {
        self.actorProperty = actorProperty
    }
    
    @MainActor func mutateProperty(_ newValue: String) {
        self.actorProperty = newValue
    }
}

print(actor.actorProperty) // ✅ now can be access it on Main thread without `await`
actor.mutateProperty("Vietnam") // ✅ it is legal 
actor.actorProperty = "Hochiminh city" // ✅ it is legal, can be mutate directly
```
--- 
🙏🏻📚 Learned from
1. https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html
2. https://www.avanderlee.com/swift/async-await/
3. https://www.avanderlee.com/swift/async-let-asynchronous-functions-in-parallel/
4. https://www.avanderlee.com/swift/asyncthrowingstream-asyncstream/
