## Copy on Write (CoW)

##### Value types - Reference types

- Value types, where each instance keeps **a unique copy of its data**, usually defined as a struct, enum, or tuple. 
- Reference types, where instances share **a single copy of the data**, and the type is usually defined as a class.
------------
CoW - The value will be copied only upon mutation, and even then, only if it has more than one reference to it

- Suppose we have 2 variables that reference to the same data.
- DELAY the copy operation until they get a change.
- Modify second variable, it will be copied and change so that only second variable is changed, first isn't.
- ***Not all value types have this behavior***, have to implement it ourselves.

```swift
func address(_ object: UnsafeRawPointer) -> String {
    let address = Int(bitPattern: object)
    return NSString(format: "%p", address) as String
}

struct Machine {
    var name: String
}

var a1 = Machine(name: "Refridge")
var a2 = a1

print(address(&a1)) // 0x10525f2d0
print(address(&a2)) // 0x10525f2e0
```
Print DIFFERENT addresses!

------------

**Implement CoW** 

```swift
struct Machine {
    private final class MachineWrapper {
        var name: String
        init(_ name: String) {
            self.name = name
        }
    }

    private var data: MachineWrapper

    var name: String {
        get { data.name }
        set {
            if !isKnownUniquelyReferenced(&data){
                self.data = MachineWrapper(newValue)
                print("Copied!!!")
            } else {
                self.data.name = newValue
            }
        }
    }

    init(name: String) {
        data = MachineWrapper(name)
    }
}
```
**Usage:**
```swift
var a1 = Machine(name: "Refridge")
var a2 = a1

a1.name = "Washing Machine"
```
Alright,
- Machine is struct, so when assigning it to another variable, **value is copied**!
- But the instance `data` inside is** a class**, so it is still **SHARED** by 2 copies UNTIL we modify the `name` of `a1` 

------------


*More*:
- [https://developer.apple.com/swift/blog/?id=10]()
- [https://stackoverflow.com/questions/43486408/does-swift-copy-on-write-for-all-structs](https://stackoverflow.com/questions/43486408/does-swift-copy-on-write-for-all-structs)
- https://jaredkhan.com/blog/swift-copy-on-write
