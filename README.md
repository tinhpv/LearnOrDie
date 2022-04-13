## Copy on Write

A little bit on reference type vs. value type
```sil
Types in Swift fall into one of two categories: first, “value types”, 
where each instance keeps a unique copy of its data, usually defined as a struct, enum, or tuple. 
The second, “reference types”, where instances share a single copy of the data, 
and the type is usually defined as a class.
```

- Value types (structs, tuples, enums) have `copy semantic`, assign to a variable or pass to a function, underlying data will be ***copied***.
- If copying a large array, Swift won't be very happy cuz it is expensive 😪

CoW:`the value will be copied only upon mutation, and even then, only if it has more than one reference to it`

- Suppose we have 2 variables that reference to the same data.
- DELAY the copy operation until they get a change.
- Modify second variable, it will be copied and change so that only second variable is changed, first isn't.
 

Note:
* Not all value types have this behavior
* Type that we create does not have it, have to implement it ourselves.


*Reference*:
[https://developer.apple.com/swift/blog/?id=10]()
