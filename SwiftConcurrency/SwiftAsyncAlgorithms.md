- [What is Swift Async Algorithms?](#what-is-swift-async-algorithms)
- [Recap AsyncSequence](#recap-asyncsequence)
- [Multi-input algorithms](#multi-input-algorithms)
  - [Zip](#zip)
  - [Merge](#merge)
- [New Clock API](#new-clock-api)
  - [Clock protocol](#clock-protocol)
  - [Built in clocks](#built-in-clocks)
    - [ContinuousClock](#continuousclock)
    - [SuspendingClock](#suspendingclock)
    - [Examples](#examples)
- [Algorithms using time](#algorithms-using-time)
  - [Debounce](#debounce)
  - [Chunks](#chunks)
- [Collections](#collections)
  - [Collection initializers](#collection-initializers)

# What is Swift Async Algorithms?

The Swift Async Algorithms package is a set of algorithms specifically focused on processing values over time using AsyncSequence. 


# Recap AsyncSequence

AsyncSequence is a protocol that lets us describe values produced asynchronously.  

Basically, it's just like Sequence, but has two key differences. 
1. The `next` function from its iterator is asynchronous, being that it can deliver values using Swift concurrency. 
2. It lets us handle any potential failures using Swift's throw effect. 

We can iterate it, using the `for-await-in` syntax. 

It takes this a step further by incorporating more advanced algorithms, as well as interoperates with clocks to give us some really powerful stuff. 

# Multi-input algorithms

They take multiple input AsyncSequences and produce one output AsyncSequence.


## Zip

The Zip algorithm takes multiple inputs and iterates them.

- Produces a tuple of the results from each of the bases.
- Iterates each of the bases concurrently 
- Rethrows errors if a failure occurs on iterating any of them. 

Zip itself has no preference on which side produced a value first or not, so a video could be produced first or a preview, and no matter which side it is, it will await for the other to send a complete tuple. 

```swift
// upload attachments of videos and previews such that every video has a preview that are created concurrently so that neither blocks each other. 

for try await (vid, preview) in zip(videos, previews) {
    try await upload(vid, preview)
}
```

## Merge

It works similarly to Zip in the regards that it concurrently iterates multiple AsyncSequences 
- Requires the bases to share the same element type
- Merges the base AsyncSequences into one singular AsyncSequence of those elements
- Takes the first element produced by any of the sides when iterated
- Keeps iterating until there are no more values that could be produced, specifically when all base AsyncSequences return nil from their iterator
- If any of the bases produces an error, the other iterations are cancelled


```swift
// Display previews of messages from either the primary or secondary account

for try await message in merge(primaryAccount.messages, secondaryAccount.messages) {
    displayPreview(message)
}
```

# New Clock API

Time itself can be a really complex subject, and new in Swift (5.7) are a set of APIs to make that safe and consistent: Clock, Instant, and Duration.

## Clock protocol

The `Clock` protocol defines two primitives
- Define a way to wake up after a given instant 
- Define a concept of now 

## Built in clocks

### ContinuousClock

Measure time just like a stopwatch, where time progresses no matter the state of the thing being measured.

### SuspendingClock

Suspends when the machine is put to sleep

### Examples

- Migrate from existing callback events to clock sleep function to handle dismissing alerts after a deadline

```swift
// Sleep until a given deadline

let clock = SuspendingClock()
var deadline = clock.now + .seconds(3)
try await clock.sleep(until: deadline)
```

- Measure the elapsed duration of execution of work

```swift
let clock = SuspendingClock()
let elapsed = await clock.measure {
  await someLongRunningWork()
}
//Elapsed time reads 00:05.40

let clock = ContinuousClock()
let elapsed = await clock.measure {
  await someLongRunningWork()
}
//Elapsed time reads 00:19.54
```

The key difference between `SuspendingClock` and `ContinuousClock` comes from its behavior when the machine is asleep.

For long running work like these, the work can be paused, just as we did here, but when we resume the execution, the ContinuousClock has progressed while the machine was asleep, but the SuspendingClock did not.  
Commonly, this difference can be the key detail to make sure things like animations work as expected by suspending the timing of the execution. 

- Interacting with time in relation to the machine, like for animations, use the `SuspendingClock`
- Measuring tasks in relation to the human in front of the device, use the `ContinuousClock`
- Delay by an absolute duration, something relative to humans, use the `ContinuousClock` 

# Algorithms using time

The Swift Async Algorithms uses these new `Clock`, `Instant`, and `Duration` types to build generic algorithms for dealing with many of the concepts of how events are processed with regards to time.

## Debounce

- Awaits a quiescence period before it emits the next values when iterated
- Rethrows failures immediately
- By default, it will use the `ContinuousClock`
-  

```swift
// Control searching messages

class SearchController {
    let searchResults = AsyncChannel<SearchResult>()

    func search<SearchValues: AsyncSequence>(_ searchValues: SearchValues) 
        where SearchValues.Element == String {

        let queries = searchValues.debounce(for: .milliseconds(300))

        for await query in queries {
            let results = try await performSearch(query)
            await channel.send(results)}
        }
    }
}
```

## Chunks

- Control over chunks by count, by time, by content
- If an error occurs in any of these, that error is rethrown

```swift
// Gather messages up tp serialize them into packets
// to be sent to the server by batching into chunks

let batches = outboundMessages.chunked(
  by: .repeating(every: .milliseconds(500))
)

let encoder = JSONEncoder() 
for await batch in batches {
  let data = try encoder.encode(batch)
  try await postToServer(data) 
}
```

We used the `chunked(by:)` API to ensure that chunks of messages are serialized and sent off by a certain elapsed duration. That way, our server gets efficient packets sent from the clients.

# Collections

`AsyncSequence` works much like how the lazy algorithms work in the Swift standard library. But just like those lazy algorithms, there are often times where we need to move back into the world of collections.

## Collection initializers

Offers a set of initializers for constructing collections using `AsyncSequence`.  `
These let us build up dictionaries, sets, or arrays with input `AsyncSequences` that are known to be finite.  
The collection initializers let us build in conversions right into our initialization of messages and keep our data types as `Array`. 

```swift
// Create a message with awaiting attachments to be encoded
init<Attachments: AsyncSequence>(_ attachments: Attachments) async rethrows {
  self.attachments = try await Array(attachments)
}
```

We have numerous features that really could use some updating to use Swift concurrency. And by keeping our existing data structures, we can migrate parts of our app incrementally and where it makes sense. 