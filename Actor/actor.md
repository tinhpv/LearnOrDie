## Actors
- Kind object type, on a par with `enum`, `class`, `struct`
- Actor is a **reference type**, similar to a `class`, can inherit from other actors, conforms to protocol, be generics, and be used with generics
- Difference is actor has *isolation domain*, which helps in protecting mutable state, prevent data races
- Actor's code run on *background* by default 

### Actor Isolation
1. If an actor instance have *mutable property* (`var`), can only be modified on `self`
2. Can access actor's **property**, but this access is *asynchronous*, must use `await`
3. Can call actor's **method**, but this access is *asynchronous*, must use `await` (even if it is not an `async` method)

Why `await`❓ To make sure, no other code belonging to the same actor can start, solves the problem of simultaneous access. With an actor, there is no simultaneous access. Access to an actor is serialized. ✅

```swift
actor BankAccount {
    let accountNumber: Int
    var balance: Double
    
    init(accountNumber: Int, balance: Double) {
        self.accountNumber = accountNumber
        self.balance = balance
    }
}

let accountA = BankAccount(accountNumber: 1, balance: 100.0)
print(accountA.accountNumber) // ✅ constant value, so can be directly accessible
print(accountA.balance) // ⛔️ mutable property, have to put await

Task {
    print(await accountA.balance) // ✅ it's legal now
}

accountA.balance = 200.0 // ⛔️ this is illegal, will trigger an error as cannot modify not on self
```

Let's add a method to the actor 👍
See, in `transfer(amount:to)`, we reference to an actor → this is called `cross-actor reference`

```swift
extension BankAccount {
    @discardableResult
    func transfer(amount: Double, to other: BankAccount) -> Bool {
        guard amount > 0 && amount <= balance else { return false }
        self.balance -= amount // ✅ legal, balance is modified on self
        other.balance += amount // illegal, not on self ➡️ modified on non-isolated actor instance
        return true
    }
}
```
**Solution:**  Add new method to the actor, say `deposit()` that legally modify `balance`
```swift
extension BankAccount {
    func deposite(_ amount: Double) {
        self.balance += amount
    }
    
    @discardableResult
    func transfer(amount: Double, to other: BankAccount) async -> Bool {
        guard amount > 0 && amount <= balance else { return false }
        self.balance -= amount
        await other.deposite(amount) // ✅ legal now
        return true
    }
}
```

**Another one:** use `isolated` keyword
`isolated` will bind method `transfer(amount:to)` to `other` isolation domain

```swift
extension BankAccount {
    @discardableResult
    func transfer(amount: Double, to other: isolated BankAccount) -> Bool {
        guard amount > 0 && amount <= balance else { return false }
        self.balance -= amount
        other.balance += amount
        return true
    }
}
```

## `Sendable` types
- Values of types that conform to `Sendable` protocol are safe to share accross concurrency domain: 
    - value-semantic type (`Int`, `Bool`, `Double`, `String`...) 
    - value-semantic collections (`[Int]`, `[Int: String]`)
    - immutable class (only containing constant value `let`)
    - class that perform its own synchronization internally
    - structure or enum contains values of type that is `Sendable` will implicitly `Sendable`
- Actor type *implicitly* conforms to `Sendable` protocol, protects its mutable state, actor instances can be safe to share accross concurrency domain.

**⚠️ Issue with Actor:**
- Add a list of owners to the actor, owner in type of `Person`
```swift
// This is not implicitly conforming to Sendable
// Since it is a class - reference type 
// have a mutable property which can be mutated on multiple threads
class Person {
    var name: String
    let age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

actor BankAccount {
    ...
    var owners: [Person]
    ...
}
```
- Add a getter function to return primary owner of the account
```swift
extension BankAccount {
    ...
    func primaryOwner() -> Person? {
        owners.first
    }
}
```
- Now, time to test...
```swift
let primaryOwnerOfAccountA = Person(name: "Aaron Phan", age: 24)
let accountA = BankAccount(accountNumber: 1, balance: 100.0, owners: [primaryOwnerOfAccountA])

Task {
    if let primaryOwner = await accountA.primaryOwner() {
        primaryOwner.name = "Phan Vu Tinh" // ⛔️ should be avoid since `name` can be mutated on multiple threads but XCode does not warn it
    }
}
```
The problem is that we still can access the actor property (which is not `Sendable`) and modify one of its properties on multiple thread without any complain

Solution:
- mark `Person` as `final` to stop further inheritance
- make it conform to `Sendable` with `@unchecked`
- encapsolate `name`, create `updateName()` to update `name` with serial queue
```swift
final class Person: @unchecked Sendable {
    private var name: String
    let age: Int
    private let serialQueue = DispatchQueue(label: "person.serial.queue")
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
    
    func updateName(_ name: String) {
        serialQueue.sync {
            self.name = name
        }
    }
}
```

## Main Actor
- Isolate code to main actor, it will run on main thread
- How? Use `@MainActor` directive on:
  - *property* (static/class/instance property) 👉 only be accessed on main thread
  - *closure*, *method* (static/class/instance property) 👉 only be called on main thread
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

⁉️ *Switching context from background thread to main thread for updating UI?*
MainActor has an extension

```swift
@available(macOS 12.0, iOS 15.0, watchOS 8.0, tvOS 15.0, *)
extension MainActor {

    /// Execute the given body closure on the main actor.
    public static func run<T>(resultType: T.Type = T.self, body: @MainActor @Sendable () throws -> T) async rethrows -> T
}
```

Usage:
```swift
let imgUrl = "https://cdn.pixabay.com/photo/2015/04/23/22/00/tree-736885__480.jpg"
Task.detached {
    // 🅱️ Run on background thread 
    do {
        let data = try await self.download(url: URL(string: imgUrl)!)
        await MainActor.run {
        // Ⓜ️ Now, switched to Main thread
            self.image1.image = UIImage(data: data)
        }
    } catch(let error) {
        print(error.localizedDescription)
    }
}
```
