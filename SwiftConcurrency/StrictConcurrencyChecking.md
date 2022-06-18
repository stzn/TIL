# Strict Concurrency Checking

- [Strict Concurrency Checking](#strict-concurrency-checking)
  - [Minimal mode](#minimal-mode)
  - [Targeted mode](#targeted-mode)
  - [Complete mode](#complete-mode)
  - [Recommendation](#recommendation)

## Minimal mode

The compiler will only diagnose places where one has explicitly tried to mark something as `Sendable`. This is similar to how Swift 5.5 and 5.6 behaved, and for the above, there won't be any warnings or errors.

For example, there are `Chicken` class in the FarmAnimals module and `Coop` which has an array of `Chicken` in the other module.

If `Chicken` is not `Sendable`, `Coop` gets a warning.

```swift
// Module: FarmAnimals

public final class Chicken { // non-Sendable
    let name: String
    init(name: String) {
        self.name = name
    }
}

import FarmAnimals

struct Coop {
    var flock: [Chicken]
}
```

Now, if we add a `Sendable` conformance, the compiler will complain that the `Coop` type cannot be `Sendable` because `Chicken` isn't `Sendable`.

```swift
import FarmAnimals

struct Coop: Sendable { // 👈🏻Need to add Sendable to turn on Sendable check
    var flock: [Chicken] // ⚠️Stored property 'flock' of 'Sendable'-conforming struct 'Coop' has non-sendable type '[Chicken]'
}
```

`Sendable`-related problems will be presented as warnings in Swift 5, not errors, to make it easier to work through the problems one by one.


## Targeted mode

This setting enables `Sendable` checking for code that has already adopted Swift Concurrency features like async/await, tasks, or actors.

For example, this will identify attempts to capture values of non-`Sendable` type in a newly created task.

```swift
import FarmAnimals

func visit(coop: Coop) async {
    guard let favorite = coop.flock.randomElement() else {
        return
    }

    Task {
        favorite.play() // ⚠️Capture of 'favorite' with non-sendable type 'Chicken' in a `@Sendable` closure
    }
}
```

Perhaps it's some package that hasn't been updated for `Sendable` yet, or even our own module that we just haven't gotten around to.

For those, we can temporarily disable the `Sendable` warnings for types that come from that module using the `@preconcurrency` attribute.

```swift
@preconcurrency import FarmAnimals // 👈🏻add @preconcurrency

func visit(coop: Coop) async {
    guard let favorite = coop.flock.randomElement() else {
        return
    }

    Task {
        favorite.play() // no warning
    }
}
```

This will silence `Sendable` warnings for the `Chicken` type within this source file.


At some point, the `FarmAnimals` module will get updated with `Sendable` conformance. Then, one of two things will happen: either `Chicken` becomes `Sendable` somehow, in which case the `preconcurrency` attribute can be removed from the import.

```swift
// Module: FarmAnimals

public final class Chicken: Sendable { ... }
```

```swift
@preconcurrency import FarmAnimals // ⚠️'@preconcurrency' attribute on module 'FarmAnimals' is unused
```

Or `Chicken` will be known to be non-`Sendable`, in which case the warning will come back, indicating that your assumptions about `Chicken` being `Sendable` are, in fact, not correct.

```swift
// Module: FarmAnimals

public final class Chicken: Sendable { ... }
```

```swift
@preconcurrency import FarmAnimals
func visit(coop: Coop) async {
    Task {
        favorite.play() // ⚠️Capture of 'favorite' with non-sendable type 'Chicken' in a `@Sendable` closure
    }
}
```

The targeted strictness setting tries to strike a balance between compatibility with existing code and identifying potential data races.

## Complete mode

It approximates the intended Swift 6 semantics to completely eliminate data races. It checks everything that the earlier two modes check but does so for all code in the module.

For example, the below code is performing work on a dispatch queue, which will execute that code concurrently.

```swift
func doWork(_ body: @escaping () -> Void) {
    DispatchQueue.global().async {
        body() // ⚠️Capture of 'body' with non-sendable type '() -> Void' in a `@Sendable` closure
    }
}

func visit(friend: Chicken) {
    doWork {
        friend.play()
    }
}
```

The async operation on a dispatch queue is actually known to take a `Sendable` closure, so the compiler produces a warning indicating that there is a data race when the non-`Sendable` body is captured by the code running on the dispatch queue.

We can fix this by making the body parameter `Sendable`. That change eliminates this warning, and now all of the callers of `doWork` know that they need to provide a `Sendable` closure.

```swift
func doWork(_ body: @Sendable @escaping () -> Void) {
    DispatchQueue.global().async {
        body() // no warning
    }
}

func visit(friend: Chicken) {
    doWork {
        friend.play() // Capture of 'friend' with non-sendable type 'Chicken' in a `@Sendable` closure
    }
}
```

Complete checking will help flush out the potential data races in our program. To achieve Swift's goal of eliminating data races, we'll eventually need to get to complete checking.

## Recommendation

- Work incrementally toward that goal: adopt Swift's concurrency model to architect our app for data race safety, then enable progressively stricter concurrency checking to eliminate classes of errors from our code.

- Don't worry about marking our imports with `@preconcurrency` to suppress warnings for imported types. As those modules adopt stricter concurrency checking, the compiler will recheck our assumptions.