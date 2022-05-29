# Async/Await

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
### Tasks
This is `a unit of asynchronous work`
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
        let result = try await URLSession.shared.data(from: url)
        return result.0 // the response data
    }
    
    ///  CONSOLE
    https://cdn.pixabay.com/photo/2015/04/23/22/00/tree-736885__480.jpg
    44304 bytes
```

**Want to adopt structured concurrency but keep old-fashioned async code based on completion handler?**

👇
### Wrapping a Completion Handler

wrapping with 
-  `withUnsafeContinuation(_:)`
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
> - `UnsafeContinuation` is basically the same kind of object that forms the basis of the `await` keyword itself. 
- when we wait for an async function call using `await`, a continuation object is created and stored behind the scenes 
- when the async function finishes, a method of that continuation object is called, and our code resumes. So what we’re doing in this wrapper method is enacting manually the same sort of wait-and-resume architecture that is performed for us automatically when we say `await`.

Learned from
1. https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html
2. https://www.avanderlee.com/swift/async-await/
