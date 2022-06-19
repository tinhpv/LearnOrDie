# App Launch Process

## Entry point
- When the app launches, system needs to know where to find the compiled binary inside app's bundle 💊 Get **name of the binary** from `Info.plist` file with `Executable file` key (`CFBundleExecutable`), whose value is usually the same as `$(PRODUCT_NAME)` Build Setting
- System locates, loads binary file and links any needed frameworks
- Where systems call into binary's code to make it run?
#### With Objective-C app → entry point is `main()`
- setup memory management environment - `@autoreleasepool {}`
- call built-in `UIApplicationMain` function to get the app running
```swift
int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    @autoreleasepool {
        // Setup code that might create autoreleased objects goes here.
        appDelegateClassName = NSStringFromClass([AppDelegate class]);
    }
    return UIApplicationMain(argc, argv, nil, appDelegateClassName);
}
```
#### with Swift app 
- notice the `@main` attribute on `AppDelegate.swift`, which does all thing that has been done on `main()` Objective-C app. 
- Can create a custom `main` function. Create new file with the name as `main.swift`:
```swift
import UIKit

UIApplicationMain(
    CommandLine.argc,
    CommandLine.unsafeArgv,
    nil,
    NSStringFromClass(AppDelegate.self)
)
```

- From Swift 5.3, can create custom struct in a file with any name, just required to give it a static function `main` and trigger `UIApplicationMain`:
```swift
import UIKit

@main
struct EntryPoint {
    static func main() {
        UIApplicationMain(
            CommandLine.argc,
            CommandLine.unsafeArgv,
            nil,
            NSStringFromClass(AppDelegate.self)
        )
    }
}
```

## Events of `UIApplicationMain` 
1. It creates shared application instance
2. Create application instance's *delegate*. See above the 4th argument, passing into `NSStringFromClass(AppDelegate.self)` specifies that `AppDelegate` will be the app delegate class.
3. Turn to app delegate, call `application(_:didFinishLaunchingWithOptions:)`, can do some initializations here
4. Create `UISceneSession`, `UIWindowScene` and app's window scene *delegate*. Entry `Application Scene Manifest` in the file `Info.plist` declare the class of the window scene delegate instance.
5. If there is storyboard defined in `Info.plist`, `UIApplicationMain` will load and find the initial view controller and the associated class, `UIApplicationMain` will initialize it
6. Create app's `window`. This window is assigned to scene delegate's `window`. Then, assign the initial view controller instance to `window`'s `rootViewController` property
7. Call scene delegate method `scene(_:willConnectTo:options:)`
8. Now, time to make UI to appear, `UIApplicationMain` calls `makeKeyAndVisible` of `UIWindow`
9. Window is about to appear. window turn to the root view controller, the main view of that VC will be obtained. If VC get its view from nib, it is loaded and its objects are instantiated and initialized.
10. Time for some methods of view controller lifecycle will be called, as `viewDidLoad()`, root view controller's main view is placed into window, which is visible to user.
 
`UIApplicationMain` has still been running, waiting for user to interact , maintain the `event loop`

