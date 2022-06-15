# UserDefaults

Preferences are grouped into **domains**, they can be **persistent** or **volatile**
- *persistent* 👉🏻 stored on memory, not losing when re-boot the app
- *viatile* 👉🏻 only valid along UserDefaults instance lifetime.

| Domain  | State |
| ------------- | ------------- |
| `NSArgumentDomain`  | volatile  |
| Application (identified by app's identifier) | persistent  |
| `NSGlobalDomain` | persistent  |
| Languages (identified by language names) | volatile  |
| `NSRegistrationDomain` | volatile  |

a preference has 3 components:
- domain in which is is stored 
- name (String)
- value (Data, String, Number, Date, Array, Dictionary)

Whenever retrieving value for a key, a search for `value` of a given `key` will go through the domain hierarchy above from top to bottom starting with `NSArgumentDomain` and return the first value it found

### Argument domain
- contains values that were set from **command line arguments** (provided when the app launched from command line)
- preferences set from the command line temporarily *override* the established values stored in the user's defaults database.

### Application domain 
- contains **app-specific preferences** that are stored in user defaults database of current user.
- preferences are automatically placed into this domain when using `UserDefaults.standard` to write.
- when the first property was written, a file with format `<bundle-identifier>.plist` will exist in `$HOME/Library/Preferences`

### Global domain
- contains preferences that are applicable to all apps
- used by system frameworks to store **system-wide values**
- should NOT be used to store app-specific data
- write the same preference on **application domain** to change value of that preference on global domain

### Languages domains
- every language in `AppleLanguages` preference will be stored in a domain whose name is that language name
- every `language-specific` domain has preferences for the corresponding locale, which is useful in some cases.


### Registration domain
- for the first time launching the app, most preferences don't have value
- used to define set of **default values** for the case a given preference has not been set
- use `registerDefaults:` of `NSUserDefaults`

ℹ️ have to do this on every launch of the app before reading any value from user defaults database since `registerDefaults:` does NOT persist the default values to memory.
ℹ️ `application:didFinishLaunchingWithOptions:` is usually the right place


📚 Learned from
- [Preferences and Settings Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/UserDefaults/AboutPreferenceDomains/AboutPreferenceDomains.html#//apple_ref/doc/uid/10000059i-CH2-SW1 "Preferences and Settings Programming Guide")
