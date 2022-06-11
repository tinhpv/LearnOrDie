# Data Race - Race Condition 🤔🆘

Data Race and Race Condition are **DIFFERENT**
- *Data-Race* → MORE than 1 thread access to shared-resource (can be a variable or object) and AT LEAST 1 thread tries to **modify** that resource
- *Race-Condition* → **Order** of threads' execution leads to wrong program behavior
- Many race conditions are due to data races, and many data races lead to race conditions. Can have race conditions without data races and data races without race conditions.

Example:
 ```swift
 @discardableResult
 func transfer(amount: Int, from source: BankAccount, to destination: BankAccount) -> Bool {
    if source.balance >= amount {
        destination.balance += amount
        source.balance -= amount
        return true
    }
    return false
}
```
🔵 **LET'S RUN ON ONE-SINGLE THREAD**
```swift
let accountA = BankAccount(balance: 100)
let accountB = BankAccount(balance: 100)

transfer(amount: 50, from: accountA, to: accountB)

print(accountA.balance) // 5️⃣0️⃣
print(accountB.balance) // 1️⃣5️⃣0️⃣️
 ```
 It makes sense! ✅
 
🔴 **LET'S RUN ON MULTIPLE THREADS**
```swift
let accountA = BankAccount(balance: 1000)
let accountB = BankAccount(balance: 2000)

...
@IBAction private func transferButtonTapped() {
    let group = DispatchGroup()
    for _ in 1...5 {
        DispatchQueue.global().async(group: group) {
            self.transfer(amount: 50, from: self.accountA, to: self.accountB)
        }
    }
    
    group.notify(queue: DispatchQueue.main) {
        self.labelAccountA.text = "\(self.accountA.balance)"
        self.labelAccountB.text = "\(self.accountB.balance)"
    }
}
...
```

We will get a warning ⛔️ in `transfer(amount:from:to)` with the help of Thread Sanitizer as 
`Data race in *.ViewController.transfer(amount: Swift.Int, from: BankAccount, to: BankAccount) -> Swift.Bool at 0x7b080002f8e0`

- ONLY mention about the ORDER of execution, which thread goes first will lead to different outcomes
- IN REALITY, while a thread is checking the condition and execute `destination.balance += amount` (we suppose), other threads is also going through these steps and unexpectedly modify the same data → the outcome is unpredictable and can even cause crash 😫

### Solutions
#### GCD to solve Data-Race
Using a serial queue, guarantee only one thread access balances
Data-race is now addressed!
⚠️ Race-condition is still able to happen, cannot manage the order of execution of threads. But it won't break the app, which is acceptable!
```swift
let serialQueue = DispatchQueue(label: "serial.queue")

@discardableResult
func transfer(amount: Int, from source: BankAccount, to destination: BankAccount) -> Bool {
    serialQueue.sync {
        if source.balance >= amount {
            destination.balance += amount
            source.balance -= amount
            return
        }
        
        return false
    }
}
```

📚 Learned from:
1. https://www.swiftbysundell.com/articles/avoiding-race-conditions-in-swift/
2. https://www.avanderlee.com/swift/race-condition-vs-data-race/
3. https://www.avanderlee.com/swift/exc-bad-access-crash/
4. https://www.avanderlee.com/swift/thread-sanitizer-data-races/
5. https://swiftsenpai.com/swift/actor-prevent-data-race/
6. https://stackoverflow.com/a/18049303
7. https://blog.regehr.org/archives/490
