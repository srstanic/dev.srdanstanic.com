---
layout: post
title:  "Swift Concurrency Quick Start"
categories: ios swift concurrency
---

If you are new to Swift Concurrency, you are in the right place. Here is everything you need to know in the most condensed form available.

The ideal audience of this post is an experienced programmer familiar with concurrency concepts and interested in learning the Swift Concurrency syntax and APIs.

Get comfy, because we have a lot of ground to cover.


Async/await syntax
------------------

Asynchronous functions are defined by adding the `async` keyword after the parenthesis in the function definition:
```swift
func loadIssues() async -> [Issue] { ... }
```

Asynchronous functions can call other asynchronous functions with the `await` keyword:
```swift
let issues = await loadIssues()
```
The `await` keyword marks a potential suspension point. The asynchronous function making this call may or may not be suspended at this point. Suspension in this context means that the function temporarily stops executing. If it is suspended, it will continue execution sometime in the future after the callee returns a result.

Asynchronous functions that can throw an error are marked with `async throws`:
```swift
func loadIssues() async throws -> [Issue] { ... }
```

Throwing asynchronous functions are called with `try await`:
```swift
let issues = try await loadIssues()
```
or with one of the other two variants: `try? await` and `try! await`.

When you want multiple asynchronous functions to execute concurrently, you assign the result of the asynchronous function call to a constant with `async let` and then wait for the results later by using `await` each time you use this constant:
```swift
func loadIssues() async -> [Issue] { ... }
func loadProjects() async -> [Project] { ... }
func loadMessages() async -> [Message] { ... }

// these three calls execute concurrently
async let issues = loadIssues()
async let projects = loadProjects()
async let messages = loadMessages()

let allData = await (issues, projects, messages)

let issuesFromAwaitedConstant = allData.0
let issuesFromAsyncConstant = await issues
```

Methods, computed properties, and subscripts can also be marked with `async` and `async throws`:
```swift
protocol IssuesStore {
  var issuesCount: Int { get async throws }
  subscript(_ index: Int) -> Issue { get async throws }
  func loadIssues() async throws -> [Issue]
}

class RemoteIssuesStore: IssuesStore {
  var issuesCount: Int {
    get async throws { ... }
  }

  subscript(_ index: Int) -> Issue {
    get async throws { ... }
  }

  func loadIssues() async throws -> [Issue] { ... }
}

let issuesStore = RemoteIssuesStore()
let issueCount = try? await issuesStore.issuesCount
let lastIssue = try? await issuesStore[issueCount! - 1]
let issues = try? await issuesStore.loadIssues()
```


Continuations
-------------

When integrating with an API that still uses completion handlers, you can use the `withCheckedContinuation(_:)` function:
```swift
func loadCurrentUser(completion: @escaping (User) -> Void) { ... }

let currentUser = await withCheckedContinuation {
  (continuation: CheckedContinuation) in
  loadCurrentUser { user in
    continuation.resume(returning: user)
  }
}
```
It's your responsibility to make sure the continuation is resumed exactly once. `CheckedContinuation` will ensure in runtime that it is resumed exactly once. If you want a lower overhead option that doesn't enforce this rule, you can use the `withUnsafeContinuation(_:)` function, which works with the `UnsafeContinuation`. `CheckedContinuation` and `UnsafeContinuation` have the same interface, so it's easy to interchange them.

If the API you are calling can return errors, you can use their throwing equivalents: `withCheckedThrowingContinuation(_:)` and `withUnsafeThrowingContinuation(_:)`.

The continuation functions can also be used to integrate with delegate-based APIs with separate success and error-handling methods. You can store the continuation object in a local property when calling the API and use it later in the result-handling delegate methods.
```swift
class Store {
  func purchaseProduct(withId productId: String) {}
  var delegate: StoreDelegate?
}

protocol StoreDelegate {
  func store(_ store: Store, didCompletePurchaseForProduct product: Product)
  func store(_ store: Store, didFailWithError error: Error)
}

class PurchaseService: StoreDelegate {
  private let store: Store
  private var continuation: CheckedContinuation<Product, Error>?

  init(store: Store) {
    self.store = store
    weak var storeDelegate = self
    store.delegate = storeDelegate
  }

  func purchaseProduct(withId productId: String) async throws -> Product {
    try await withCheckedThrowingContinuation { [weak self] continuation in
      self?.continuation = continuation
      store.purchaseProduct(withId: productId)
    }
  }

  // MARK: StoreDelegate

  func store(_ store: Store, didCompletePurchaseForProduct product: Product) {
    continuation?.resume(returning: product)
  }

  func store(_ store: Store, didFailWithError error: Error) {
    continuation?.resume(throwing: error)
  }
}
```


Structured and unstructured tasks
---------------------------------

Asynchronous functions cannot be called from synchronous functions:
```swift
func doSomeAsyncWork() async { ... }

func doSomeWork() {
  await doSomeAsyncWork() // Compilation error: 'async' call in a function that does not support concurrency
}
```

To call an asynchronous function, we need to use `await`. Since `await` requires the program to be able to suspend execution, it can only be used in an asynchronous function.

So how do we enter the initial asynchronous function? Here are the available options:
- The static `main()` method of a structure, class, or enumeration that's marked with `@main` (for command-line programs)
- SwiftUI view modifiers `.task()` and `.refreshable()`
- An unstructured task

Let's unpack that third option.

A task is a unit of asynchronous work. All asynchronous functions run as part of some task. A task can run only one function at a time, but a series of function calls can be executed in one task. When an asynchronous function calls another asynchronous function with `await`, the called function still runs as part of the same task. When the called function returns, the calling function continues running in that same task until it also returns.

A task can have one or more child tasks. Child tasks can run concurrently with the parent task. A program can consist of a complex tree of tasks. It can be hard to properly propagate cancellations and ensure no loose child tasks are left behind when a parent task completes. That's why **structured concurrency** was introduced.

Structured concurrency is an explicit parent-child relationship that allows the Swift compiler to raise errors upfront and Swift runtime to help us manage the tasks. Under structured concurrency, a child task must complete before the parent task completes, and if a parent task throws an error, all child tasks are canceled.

Cancellation is cooperative, so if the child tasks do not comply, the parent task will be blocked and won't return until all child tasks complete. There is no way for the parent task to force the child task to complete.

### Top-level task

Every task tree has a top-level task. When we use SwiftUI modifiers to run asynchronous code or the `@main` annotation in command line programs, Swift creates a top-level task for us. But we can create and run a top-level task ourselves by initializing a `Task`:
```swift
func doSomeAsyncWork() async { ... }

func doSomeWork() {
  Task {
    await doSomeAsyncWork()
  }
}
```
This task is **unstructured** because it doesn't have a parent task. We can hold a reference to this task and cancel it later if the state of the app changes so that we don't need the task to complete:
```swift
func doSomeWork() {
  let task = Task {
    await doSomeAsyncWork()
  }
  task.cancel()
}
```
Unless it is cancelled, an unstructured task will outlive the context in which it was created and run until completion independently, even if we don't hold a reference to it.

We can't return any values from a task created in the synchronous code since the synchronous code cannot use a suspension point to wait for the result.
```swift
func doSomeAsyncWork() async -> String { ... }

func doSomeWork() {
  let task = Task {
    await doSomeAsyncWork()
  }
  let result = await task.value // Compilation error: 'async' property access in a function that does not support concurrency
}
```
But we can capture references from the synchronous code and modify their state within the task.
```swift
class Label {
  var text: String?
}

var label = Label()

func doSomeWork() {
  Task {
    let result = await doSomeAsyncWork()
    label.text = result
  }
}
```

We can create a new `Task` within a `Task`:
```swift
func doSomeWork() {
  Task {
    Task { ... }
  }
}
```
Although this may look like the inner task is a child task of the outer task, that is not the case. These are two unrelated top-level tasks; it just so happens that one creates the other.

We can create a new task within an `async` function:
```swift
func doSomeAsyncWork() async {
  Task { ... }
}
```
As we established earlier, the `async` function is already running as part of a task, but the task created in the function is not a child task of the task running this function. It is, again, a new and unrelated top-level task.

Whenever a `Task` is created, it is a top-level task, regardless if it is created in a synchronous or asynchronous context. That is what makes it unstructured.


### Child tasks

An unstructured task is simply a root node of a structured concurrency tree. Any child tasks created under a top-level task fall under structured concurrency rules.

There are two ways to create child tasks:
1. `async let`
2. `TaskGroup` API

Between the two options, `async let` is less verbose, but `TaskGroup` API allows dynamic creation of child tasks and some control over their priority and cancellation.

With the `async let` expression, a child task is created to run the called function:
```swift
func loadIssues() async -> [Issue] { ... }

func useData() async {
  async let issues = loadIssues() // creates a child task
  ...
}
```

With the `TaskGroup` API, we can create a dynamic number of child tasks:
```swift
struct Document {}

func fetchDocument(from url: URL) async -> Document { ... }

func fetchDocuments(from urls: [URL]) async -> [Document] {
  await withTaskGroup(of: Document.self) { taskGroup in
    for url in urls {
      taskGroup.addTask {
        await fetchDocument(from: url)
      }
    }
    let documents = await taskGroup.reduce([]) { partialResult, document in
      partialResult + [document]
    }
    return documents
  }
}
```
Each child task in the task group returns the same type - one provided as an argument to the `withTaskGroup(of:body:)` function. In the example provided, it is a `Document`. You can use an enum to support different results from individual child tasks.

The task group can return a different result from the individual child tasks. In our example, it is a `Document` array.

Let's take a look at how we collected the results from the task group:
```swift
let documents = await taskGroup.reduce([]) { partialResult, document in
  partialResult + [document]
}
```
A `TaskGroup` is an `AsyncSequence`. An `AsyncSequence` is like a regular `Sequence` but we iterate over its elements asynchronously:
```swift
for await element in sequence {
  ...
}
```
Asynchronous sequences also support many higher-order functions, and that is why we could use `reduce()` in the earlier example to collect the results.

The order in which the child tasks complete defines the order in which the results will appear in the sequence - the fastest task will produce the first element of the sequence, and the slowest task will produce the last element.

To create child tasks that can throw an error, use the `withThrowingTaskGroup(of:body:)` function.

<div class="ml-embedded embedded-email-form" data-form="zRlsTe"></div>


Task execution control
----------------------

Both structured and unstructured tasks have all the control over their execution - they can choose to suspend temporarily or to end execution at some point after they are cancelled.

A task can use `Task.isCancelled` and `Task.checkCancellation()` to check if it is cancelled and decide if it wants to continue executing at different stages of its operation.
```swift
Task {
  // do some work
  if Task.isCancelled {
    return
  }
  // do some additional work
}

// or

Task {
  // do some work
  try Task.checkCancellation() // throws `CancellationError` if the task is cancelled
  // do some additional work
}
```

A long-running task can voluntarily suspend itself by calling `Task.yield()` so that Swift can give other tasks a chance to make progress with their work:
```swift
Task {
  for intenseWorkItem in arrayOfIntenseWorkItems {
    doWork(from: intenseWorkItem)
    await Task.yield()
  }
}
```

A task can also temporarily stop progress by calling `Task.sleep()` which suspends it for at least a given number of nanoseconds:
```swift
Task {
  try await Task.sleep(nanoseconds: 1_000_000_000) // sleeps for a second
}
```
A sleeping task will resume progress as soon as it gets cancelled. This may sound unexpected, but bear in mind that the cancellation is cooperative in Swift, so the task is woken up on cancellation, and it's up to the task to complete early if it wants to:
```swift
let task = Task {
  // some work
  do {
    try await Task.sleep(nanoseconds: 1_000_000_000) // sleeps for a second
  } catch {
    let cancellationError = error as? CancellationError // cancellationError is not nil
    try Task.checkCancellation() // will complete the task early with CancellationError
  }
  // some more work
}
task.cancel()
```


Task priority
-------------

Tasks can have different priorities that affect the order in which they run. We can specify a priority for a top-level task in the initializer and inspect it within the task:
```swift
Task.init(priority: .high) {
  print(Task.currentPriority) // prints "TaskPriority(rawValue: 25)"
  Task.init(priority: .low) {
    print(Task.currentPriority) // prints "TaskPriority(rawValue: 17)"
  }
}
```
The value of `Task.currentPriority` may differ from the one assigned to the task on creation if the system escalates the task's priority to address priority inversion. It's best to treat the priority as a suggestion for the Swift runtime and not to have expectations of what the value of the `Task.currentPriority` will be.

Child tasks inherit the priority of their parent task. It's also possible to define a priority for a child task with the `TaskGroup` API, but based on my experiments and [forum](https://forums.swift.org/t/priority-of-child-tasks/57865/){:target="_blank"} [discussions](https://forums.swift.org/t/task-priority-elevation-for-task-groups-and-async-let/61100){:target="_blank"}, the resulting priority can be unexpected. The [Swift Evolution Proposal](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#creating-taskgroup-child-tasks){:target="_blank"} recommends not specifying the priority manually for child tasks.


Actors
------

If there is a shared state in the program that can be accessed concurrently and needs to be protected, we can use an `Actor` type:
```swift
actor Inbox {
  private (set) var messages: [Message]
  func markMessageAsRead(at index: Int) { ... }
}
```
Actors are like classes but without inheritance. They protect their mutable state with actor isolation. Actor isolation is a collection of limitations on how the actors can be used. In effect, only one task can modify the actor's state at any moment. This is ensured by requiring the usage of `await` with access to actor-isolated properties and methods:
```swift
let inbox = Inbox()
if await !inbox.messages.isEmpty {
  await inbox.markMessageAsRead(at: 0)
}
```
All methods, subscripts, computed properties, and mutable properties are actor-isolated by default. If they are not accessing the mutable state, you can allow synchronous access with the `nonisolated` keyword:
```swift
struct User {
  let name: String
}

actor Inbox {
  let refreshRate: TimeInterval
  private let user: User
  ...
  nonisolated var userName: String {
    user.name
  }
}

let inbox = Inbox()
print(inbox.refreshRate)
print(inbox.userName)
```
Immutable properties are not actor-isolated and can be accessed synchronously.

Actor-isolated methods are [reentrants](https://forums.swift.org/t/accepted-with-modification-se-0306-actors/47662#reentrancy-4){:target="_blank"}. This means that although only one task can execute isolated methods at any given moment, it is possible for method executions to [interleave](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md#interleaving-execution-with-reentrant-actors){:target="_blank"} at suspension points. It is best to design actors with synchronous methods working on mutable state and small asynchronous methods calling these synchronous methods.


### Global Actors and the Main Actor

Actors give us concurrent access protection for the state of an individual instance, but sometimes we need this protection for the global state or external resources. Global actors address this need.

A global actor is a globally-unique actor identified by a type. This type is also a custom attribute that can be applied to any declaration. A declaration marked with this attribute is then isolated to this actor, which ensures that only one task may access this declaration at a time.

The primary motivation for introducing global actors is the need to represent the main thread within Swift Concurrency. `MainActor` is a global actor which ensures that any declaration marked with `@MainActor` is accessed from the main thread:
```swift
@MainActor
func updateUI() { ... }
```
Additionally, because of global actor inference, declarations that aren't directly marked with `@MainActor` can also be isolated to the `MainActor` unless they are explicitly annotated with a global actor or `nonisolated`. Swift Evolution Proposal for global actors specifies all the [global actor inference rules](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md#global-actor-inference){:target="_blank"}.


### Swift Concurrency and Threads

You probably noticed I hadn't mentioned threads until the global actors section. That was intentional. Swift Concurrency provides a set of abstractions and APIs which hide the thread management from the developer. And in most cases, the developer doesn't need to think about threads and how they are managed under the hood. The `MainActor` is an exception. So let's take a quick look at how tasks and threads work together.

Swift manages a group of threads called the *cooperative thread pool*. This thread pool can have as many threads as there are CPU cores multiplied by the number of task priorities. But no more than that. (No [thread explosion](https://developer.apple.com/videos/play/wwdc2015/718/?time=2027){:target="_blank"} possible under Swift Concurrency, yay!)

Tasks are scheduled to be executed on the cooperative threads. Tasks can run several asynchronous calls in a row until they get blocked by an infrastructure call or a busy actor. At that point, the task is suspended, and another task gets a chance to run on this thread. There is no guarantee that the code before and after the suspension point will run on the same thread, so later, when the task is unblocked and gets to run again, it might run on a different thread.

Like all asynchronous code, asynchronous `Actor` methods are executed within a task. A task will run a series of asynchronous actor calls on the same thread as long as it is not suspended for waiting on a result. Switching between actors is called *actor hopping*, and it is generally very efficient since there is no thread switching under the hood. That is unless the `MainActor` is involved.

The main thread is not a part of the cooperative thread pool, and actor hopping involving the `MainActor` requires a thread context switch:
1. If the asynchronous code is running isolated to the `MainActor`, and on the main thread as a result, when making an asynchronous call to a different actor, the task will be suspended and scheduled to run on the cooperative thread.
2. If actor-isolated code is running in a task on the cooperative thread and making an asynchronous call isolated to the `MainActor`, this will result in the task being suspended and scheduled to run on the main thread.

This behavior can cause performance issues if we are not being careful. Let's take a look at an example:
```swift
actor Incrementor {
  private var counter: Int = 0

  func increment() async -> Int {
    counter += 1
    return counter
  }
}

class IncrementingViewController: UIViewController {
  private let label = UILabel()
  private let incrementor = Incrementor()

  override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)
    Task { [weak self] in
      for i in 0..<100 {
        let count = await self?.increment() // runs on the cooperative thread
        self?.label.text = "\(count!)" // runs on the main thread
      }
    }
}
```
This example illustrates a real-life situation where you have hundreds of items to update on the screen in a short amount of time, and the data retrieval happens on a non-main actor. The implementation shown in the example makes 200 context switches, which is a significant overhead. Having frequent context switches reduces the performance of the program. Try to avoid repeated actor hopping involving the `MainActor`.


### Leaving the Main Actor

The flow of control in an application starts on the `MainActor`, and if you want to move the work off the `MainActor` immediately, there are two ways to do that:
1. make an actor call
2. create a detached task

Unlike a `Task` created with the initializer, which inherits the actor context, a detached task does not:
```swift
Task.detached {
  // runs on the cooperative thread
}
```


Sendable
--------

With Swift Concurrency, we have two ways to keep data safe from concurrent access: tasks and actors. The task's local scope and actor-isolated declarations are safe areas for mutable state. There is no way for this state to be accessed concurrently. We call these areas *concurrency domains* or *isolation domains*.

Data can cross isolation domains when:
1. a task receives values through arguments, captures values from the outer scope, or returns values
2. an actor method receives values through arguments or returns values

If the mutable state is passed across isolation domains, there is a risk for data races.

Swift Concurrency provides compiler support to ensure only safe data is passed across isolation domains. Types implementing the `Sendable` protocol and functions marked with the `@Sendable` attribute are considered safe.
```swift
// Sendable protocol conformance
final class Post: Sendable { ... }

// Sendable function
@Sendable func does(_ string: String, occurIn post: Post) -> Bool { ... }

// Sendable closure
let footer: String = "..."
let appendFooter = { @Sendable (post: String) -> String in
  post + footer
}
```

The following types can be marked as sendable:
- value types
- reference types with no mutable storage
- reference types that internally manage access to their state
- functions and closures

The `Sendable` protocol doesn't have any required properties or methods, but it does have semantic requirements that ensure no mutable data is involved. Full requirements are defined in the [Sendable documentation](https://developer.apple.com/documentation/swift/sendable){:target="_blank"}.

If a non-`Sendable` type is passed across isolation domains, the compiler will raise warnings in Swift 5.x, which will become errors in Swift 6.

It's possible to declare conformance to the Sendable protocol without compiler enforcement:
```swift
final class Post: @unchecked Sendable { ... }
```
With `@unchecked Sendable`, the developer is responsible for protecting the mutable state within this type by using available access synchronization techniques, such as queues and locks.


Closing thoughts
----------------

Alright. That was a lot. Swift Concurrency is no trivial topic. It brings many new and powerful APIs to master. There is more to learn than we covered here, but I hope this gives you a head start.

If you have any questions or inaccuracies to report, please let me know [@srstanic](https://twitter.com/srstanic){:target="_blank"}. Below you can find all the references I've used for this article.


References
----------

- [https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/){:target="_blank"}
- [https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md){:target="_blank"}
- [https://github.com/apple/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md](https://github.com/apple/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md){:target="_blank"}
- [https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md){:target="_blank"}
- [https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md){:target="_blank"}
- [Meet async/await in Swift (WWDC21)](https://developer.apple.com/videos/play/wwdc2021/10132){:target="_blank"}
- [Swift concurrency: Behind the scenes(WWDC21)](https://developer.apple.com/videos/play/wwdc2021/10254/){:target="_blank"}
- [Eliminate data races using Swift Concurrency (WWDC22)](https://developer.apple.com/videos/play/wwdc2022/110351){:target="_blank"}
- [Building Responsive and Efficient Apps with GCD (WWCD15)](https://developer.apple.com/videos/play/wwdc2015/718){:target="_blank"}
- [https://www.hackingwithswift.com/quick-start/concurrency/](https://www.hackingwithswift.com/quick-start/concurrency/){:target="_blank"}
- [https://swiftsenpai.com/swift/swift-concurrency-prevent-thread-explosion/](https://swiftsenpai.com/swift/swift-concurrency-prevent-thread-explosion/){:target="_blank"}
- [https://forums.swift.org/t/task-priority-elevation-for-task-groups-and-async-let/61100/](https://forums.swift.org/t/task-priority-elevation-for-task-groups-and-async-let/61100/){:target="_blank"}
- [https://forums.swift.org/t/priority-of-child-tasks/57865/](https://forums.swift.org/t/priority-of-child-tasks/57865/){:target="_blank"}
