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
## Tasks
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
## Wrapping a Completion Handler

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
A cleaner code 🐣
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

Learned from
1. https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html
2. https://www.avanderlee.com/swift/async-await/
3. https://www.avanderlee.com/swift/async-let-asynchronous-functions-in-parallel/
