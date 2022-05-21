PROBLEM:
```swift
class ProductViewController: UIViewController {
    private lazy var buyButton = Button()
    private let purchaseController: PurchaseController
    
    ...

    override func viewDidLoad() {
        super.viewDidLoad()

        // Since our buyButton retains its closure, and our
        // view controller in turn retains that button, we'll
        // end up with a retain cycle by capturing self here:
        buyButton.handler = {
            self.showShoppingCart()
            self.purchaseController.startPurchasingProcess()
        }
    }

    private func showShoppingCart() {
        ...
    }
}
```

There are two main reasons `weak` is useful:
- To prevent retain cycles.
- To prevent objects living longer than they should be.

##### Weak vs Unowned
- `weak` is declared as an Optional while `unowned` is not. 
- By declaring it `weak` you get to handle the case that it might be nil inside the closure at some point. 
- If you try to access an `unowned` variable that happens to be nil, it will crash the whole program.
- Only use `unowned` when you are positive that variable will always be around while the closure is around.

##### Escaping closure - Non-escaping closure
- @escaping closure, can be stored, passed around and executed later.
- @nonescaping closure, executed in scope, immediately, can NOT be stored and run later

##### Retain cycles
Nonescaping closure > NO retain cycles!
Escaping closure, potentially have retain cycles once:
- Closure is stored in a property or passed to another closure
- Object inside that closure maintains a strong reference to the closure (for example, `self`)

##### Delay deallocation
- By default, closure capture strongly all objects in its body (e.g. `self`) 
- Memory of these objects will not be cleaned up while closure is STILL **ALIVE**
- Scenarios that keep the closure scope alive:
    * A closure (escaping or non-escaping) might be doing some expensive serial operations, thereby delaying its scope from returning until all the work is completed…
    * An escaping closure waits for a callback with a long timeout.
    * A closure (escaping or non-escaping) might employ some thread blocking mechanism (such as DispatchSemaphore) that can delay or prevent its scope from returning
    * An escaping closure might be scheduled to execute after a delay (e.g. `DispatchQueue.asyncAfter` or `UIViewPropertyAnimator.startAnimation(afterDelay:)`)

Example:
```swift
func delayedAllocAsyncCall() {
    let url = URL(string: "https://www.google.com:81")!

    let sessionConfig = URLSessionConfiguration.default
    sessionConfig.timeoutIntervalForRequest = 999.0
    sessionConfig.timeoutIntervalForResource = 999.0
    let session = URLSession(configuration: sessionConfig)

    let demoTask = session.downloadTask(with: url) { localURL, _, error in
        guard let localURL = localURL else { return }
        let contents = (try? String(contentsOf: localURL)) ?? "No contents"
        print(contents)
        print(self.view.description)
    }
    demoTask.resume()
}
```
What's going on?
- `demoTask` run immediately, not be stored anywhere else in the class.
- the request got the time out of 999s
- completion handler of `downloadTask`does NOT cause retain cycle.
- BUT... 
    - while running the app, downloadTask is running but is not completed... but we *dismiss* the view controller
    - the closure strongly capture `self` (line 13) ⇢ its memory will not be cleaned.
    - **SOLUTION**: use `[weak self]` here... DO NOT use `[unowned self]` as the app get crashed.

Another example:
```swift
func process(image: UIImage, completion: @escaping (UIImage?) -> Void) {
    DispatchQueue.global(qos: .userInteractive).async { [weak self] in
        guard let self = self else { return }
        // perform expensive sequential work on the image
        let rotated = self.rotate(image: image)
        let cropped = self.crop(image: rotated)
        let scaled = self.scale(image: cropped)
        let processedImage = self.filter(image: scaled)
        completion(processedImage)
    }
}
```
- line 5, 6, 7 do serial expensive operations.
- use `[weak self]` but use `guard let` to unwrap `self`
    - will create *temporary* strong reference to `self` within the closure lifecycle.
    - will prevent `self` from deallocating until the closure is completedly run
    - **SOLUTION**: use **optional chaining** instead. it will check nil for `self` in every method call at line 5, 6, 7, if self == nil, skip!


###### Timers
```swift
func leakyTimer() {
    let timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { timer in
        let currentColor = self.view.backgroundColor
        self.view.backgroundColor = currentColor == .red ? .blue : .red
    }
    timer.tolerance = 0.1
    RunLoop.current.add(timer, forMode: RunLoop.Mode.common)
}
```
- it REPEATS
- closure does NOT use capture list `[weak self]` > strongly capture `self`
➡ delay deallocation
- so, use `[weak self]` here and invalidate the timer when it is no longer needed.

##### Store a function in a variable
```swift
class FirstViewController: UIViewController {
  var closure: (() -> Void)?
}

class SecondViewController: UIViewController {
  var firstVC = FirstViewController()

  func setupClosure() {
    firstVC.closure = printer
  }

  func printer() {
    print(self.view.description) 
  }
}
```

- This causes retain cycle!
- line 9, closure strongly capture `self` (`self.printer`) and `self` strongly reference to `FirstViewController` that own the closure
- ** SOLUTION:**

```swift
  func setupClosure() {
    firstVC.closure = { [weak self] in
        self?.printer()
    }
  }
```

##### Nested closures
capture list `[weak self]` once in the outer block is enough. Else, if not using capture list here, retain cycle is made.
```swift
parentFunction { [weak self] in 
    self?.childFunction {
        self?.foo()
    }
}
```

If use `guard let self = self` inside, have to capture list `[weak self]` to the inner block
```swift
parentFunction { [weak self] in 
    guard let self = self else { return }
    self.childFunction { [weak self] in // need another here
        self?.foo()
    }
}
```

Full example:
```swift
class Experiment {
    
    var _someFunctionWithTrailingClosure: (() -> ())?
    var _anotherFunctionWithTrailingClosure: (() -> ())?
    
    func someFunctionWithTrailingClosure(closure: @escaping () -> Void) {
        print("starting", #function)
        _someFunctionWithTrailingClosure = closure
        
        // Nothing happens as retain cycle if no `[weak self]` here
        // But it will cause delayed allocation, self will live until the block completes.
        DispatchQueue.main.asyncAfter(deadline: .now() + 5) { [weak self] in
            self?._someFunctionWithTrailingClosure?()
            print("finishing", #function)
        }
    }

    func anotherFunctionWithTrailingClosure(closure: @escaping () -> Void) {
        print("starting", #function)
        _anotherFunctionWithTrailingClosure = closure
        
        DispatchQueue.main.asyncAfter(deadline: .now() + 5) { [weak self] in
            self?._anotherFunctionWithTrailingClosure?()
            print("finishing", #function)
        }
    }

    func doSomething() {
        print(#function)
    }

    func testCompletionHandlers() {
        someFunctionWithTrailingClosure { [weak self] in
            guard let self = self else { return }
            self.anotherFunctionWithTrailingClosure { [weak self] in
                self?.doSomething()
            }
        }
    }

    // to track when the object is deallocated
    // if `deinit` does not get called, retain cycle is happened.
    deinit {
        print("deinit")
    }
}

func performExperiment() {
    DispatchQueue.global().async {
        let obj = Experiment()
        obj.testCompletionHandlers()

        // sleep long enough for `anotherFunctionWithTrailingClosure` to start, but not yet call its completion handler
        Thread.sleep(forTimeInterval: 7)
    }
}

performExperiment()
```

##### Grand Dispatch Queue
```swift
func nonLeakyDispatchQueue() {
    DispatchQueue.main.asyncAfter(deadline: .now() + 1.0) {
        self.view.backgroundColor = .red
    }

    DispatchQueue.main.async {
        self.view.backgroundColor = .red
    }

    DispatchQueue.global(qos: .background).async {
        print(self.navigationItem.description)
    }
}
```
- Do NOT cause retain cycle, even when the closure strongly capture `self`
- Because `self` does not reference `DispatchQueue.main`
- Only when store them to variables that belong to `self`

```swift
func leakyDispatchQueue() {
    let workItem = DispatchWorkItem { self.view.backgroundColor = .red }
    DispatchQueue.main.asyncAfter(deadline: .now() + 1.0, execute: workItem)
    self.workItem = workItem // stored in a property
}
```

##### Alternatives to `weak`
- Define a tuple which contains only objects needed for the closure
```swift
let context = (
    parser: parser,
    schema: schema,
    titleLabel: titleLabel,
    textLabel: textLabel
)

dataLoader.loadData(from: url) { data in
    // We can now use the context instead of having to capture 'self'
    let model = try context.parser.parse(data, using: context.schema)
    context.titleLabel.text = model.title
    context.textLabel.text = model.text
}
```



Learned from:
- https://stackoverflow.com/a/62352667
- https://stackoverflow.com/a/41992442
- https://medium.com/@almalehdev/you-dont-always-need-weak-self-a778bec505ef
- https://www.avanderlee.com/swift/weak-self/
- https://www.swiftbysundell.com/articles/capturing-objects-in-swift-closures/
- https://www.swiftbysundell.com/questions/is-weak-self-always-required/
- https://alisoftware.github.io/swift/closures/2016/07/25/closure-capture-1/
