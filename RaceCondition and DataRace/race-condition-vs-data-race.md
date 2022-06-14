# Data Race - Race Condition 🤔🆘

Data Race and Race Condition are **DIFFERENT**
- *Data-Race* → MORE than 1 thread access to shared-resource (can be a variable or object) and AT LEAST 1 thread tries to **modify** that resource
- *Race-Condition* → **Order** of threads' execution leads to wrong program behavior
- Many race conditions are due to data races, and many data races lead to race conditions. Can have race conditions without data races and data races without race conditions.

Example:
 ```swift
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
let firstBank = BankAccount(balance: 100)
let secondBank = BankAccount(balance: 100)
transfer(amount: 50, from: firstBank, to: secondBank)

print(firstBank.balance) // 5️⃣0️⃣
print(secondBank.balance) // 1️⃣5️⃣0️⃣️
 ```
 It makes sense! ✅
 
🔴 **LET'S RUN ON MULTIPLE THREADS**
```swift
let firstBank = BankAccount(balance: 100)
let secondBank = BankAccount(balance: 100)
transfer(amount: 50, from: firstBank, to: secondBank) // run on thread (1)
transfer(amount: 70, from: firstBank, to: secondBank) // run on thread (2)

// if (1) goes first
print(firstBank.balance) // 5️⃣0️⃣
print(secondBank.balance) // 1️⃣5️⃣0️⃣️

// but if (2) goes first
print(firstBank.balance) // 3️⃣0️⃣
print(secondBank.balance) // 1️⃣7️⃣0️⃣️
```
- ONLY mention about the ORDER of execution, (1) goes first or (2) goes first lead to 2 different outcomes
- IN REALITY, while (1) is checking the condition and execute `destination.balance += amount` (we suppose), thread (2) is also going through these steps and unexpectedly modify the same data → the outcome is unpredictable and can even cause crash 😫

📚 Learned from:
1. https://www.swiftbysundell.com/articles/avoiding-race-conditions-in-swift/
2. https://www.avanderlee.com/swift/race-condition-vs-data-race/
3. https://www.avanderlee.com/swift/exc-bad-access-crash/
4. https://www.avanderlee.com/swift/thread-sanitizer-data-races/
5. https://swiftsenpai.com/swift/actor-prevent-data-race/
6. https://stackoverflow.com/a/18049303
7. https://blog.regehr.org/archives/490
