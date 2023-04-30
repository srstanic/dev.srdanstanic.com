---
layout: post
title:  "How to add logging to an iOS app in a clean way"
categories: ios architecture logging clean
---

Imagine we are working on a banking app. This iOS app allows users to access their bank account and view a list of transactions.

Since our app needs to be fast and reliable, we've decided to log the time it takes to load transactions.

Transactions list before logging
--------------------------------

Our feature code is comprised of a view model, an abstract data-loading interface, and a data-loading implementation.
<img src="{{ site.url }}/assets/ios-application-logging-clean/no-logging.png" class="post-content-image centered" title="No Logging">

The view model uses the loader to load the data, maps the result to a presentation model, and publishes the new value.
```swift
// Feature

class TransactionsListViewModel {
  @Published var transactionItems = [TransactionsListItemViewContent]()

  private let transactionsLoader: TransactionsLoading

  init(transactionsLoader: TransactionsLoading) {
    self.transactionsLoader = transactionsLoader
  }

  func loadTransactions() async {
    let transactions = await transactionsLoader.loadTransactions()
    transactionItems = transactions.map(TransactionsListItemViewContent.map)
  }
}

// Composition

let loader = RemoteTransactionsRepository()
let viewModel = TransactionsListViewModel(transactionsLoader: loader)

// View layer triggers the load

await viewModel.loadTransactions()
```

Let's add logging
-----------------

The most common way to add logging is to have a global singleton logger instance and reference it directly in the feature code.

<img src="{{ site.url }}/assets/ios-application-logging-clean/logging-global-singleton.png" class="post-content-image centered" title="Logging with a global singleton">

Here is what this looks like in the code:
```swift
// Logger

class Logger {
  private init() {}
  static let sharedInstance: Logger = .init()
  func log(_ message: String) { print("debug: \(message)") }
}

// Date helper to calculate the passed time

extension Date {
  var timeIntervalSince: TimeInterval {
    Date().timeIntervalSince(self)
  }
}

// Feature

class TransactionsListViewModel {
  //...

  func loadTransactions() async {
    let startTime = Date()
    let transactions = await transactionsLoader.loadTransactions()
    let loadingTime = Date().timeIntervalSince(startTime)
    Logger.sharedInstance.log("Transactions loading time: \(loadingTime)")
    transactionItems = transactions.map(TransactionsListItemViewContent.map)
  }
}
```

In this article, we are more concerned about how to integrate logging into the feature than we are about the implementation details of the logging system. With that in mind, our logger implementation simply prints the message to the console.

Moving our focus to the logging integration point, we notice several drawbacks:
1. If we want to replace the logger type with a different one, we need to modify the feature code.
1. If we decide to introduce a 3rd party library for logging, our feature code will directly depend on it.
1. If there is a requirement to turn off logging for this use case based on context (e.g., tests), it's impossible to implement it without adding further complexity to the feature code.

Logging within the feature code and depending on a specific logger type makes this design very rigid.

Logger injection
----------------

One way to address the direct dependency concern is to hide the specific logger type behind an interface and inject it through the initializer.

<img src="{{ site.url }}/assets/ios-application-logging-clean/logger-injection.png" class="post-content-image centered" title="Logging injection">


Here is what this looks like in the code:
```swift
// Logger

protocol Logging {
  func log(_ message: String)
}

extension Logger: Logging {}

// Feature

class TransactionsListViewModel {
  @Published var transactionItems = [TransactionsListItemViewContent]()

  private let transactionsLoader: TransactionsLoading
  private let logger: Logging

  init(transactionsLoader: TransactionsLoading, logger: Logging) {
    self.transactionsLoader = transactionsLoader
    self.logger = logger
  }

  func loadTransactions() async {
    let startTime = Date()
    let transactions = await transactionsLoader.loadTransactions()
    logger.log("Transactions loading time: \(startTime.timeIntervalSince)")
    transactionItems = transactions.map(TransactionsListItemViewContent.map)
  }
}

// Composition

let loader = RemoteTransactionsRepository()
let logger = Logger.sharedInstance
let viewModel = TransactionsListViewModel(transactionsLoader: loader, logger: logger)
```

This design abstracts the logger type and removes a direct dependency on any logging-related libraries. It is certainly an improvement since we can easily replace the logger implementation with a different one. But the feature code is still polluted with a cross-cutting concern.

Logging Decorator
-----------------

We can use a decorator pattern to remove the logging from the feature code altogether. With the following design, the feature code is unaffected and unaware of the application's logging capabilities.

<img src="{{ site.url }}/assets/ios-application-logging-clean/logging-decorator.png" class="post-content-image centered" title="Logging decorator">

Here is what this looks like in the code:

```swift
// Feature

class TransactionsListViewModel {
  @Published var transactionItems = [TransactionsListItemViewContent]()

  private let transactionsLoader: TransactionsLoading

  init(transactionsLoader: TransactionsLoading) {
    self.transactionsLoader = transactionsLoader
  }

  func loadTransactions() async {
    let transactions = await transactionsLoader.loadTransactions()
    transactionItems = transactions.map(TransactionsListItemViewContent.map)
  }
}

// Logging

class TransactionsLoadingLoggingDecorator: TransactionsLoading {
  private let decoratee: TransactionsLoading

  init(decoratee: TransactionsLoading) {
    self.decoratee = decoratee
  }

  func loadTransactions() async -> [Transaction] {
    let startTime = Date()
    let transactions = await decoratee.loadTransactions()
    Logger.sharedInstance.log("Transactions loading time: \(startTime.timeIntervalSince)")
    return transactions
  }
}

// Composition

let loader = RemoteTransactionsRepository()
let loaderLoggingDecorator = TransactionsLoadingLoggingDecorator(decoratee: loader)
let viewModel = TransactionsListViewModel(transactionsLoader: loaderLoggingDecorator)

```

If you prefer a bit more functional design, here is what that would look like:


```swift
// Feature

typealias LoadTransactions = () async -> [Transaction]

class TransactionsListViewModel {
  @Published var transactionItems = [TransactionsListItemViewContent]()

  private let loadTransactions: LoadTransactions

  init(loadTransactions: @escaping LoadTransactions) {
    self.loadTransactions = loadTransactions
  }

  func loadTransactions() async {
    let transactions = await loadTransactions()
    transactionItems = transactions.map(TransactionsListItemViewContent.map)
  }
}

// Logging

func decorateWithLogging(
  loadTransactions: @escaping LoadTransactions
) -> LoadTransactions {
  return {
    let startTime = Date()
    let transactions = await loadTransactions()
    Logger.sharedInstance.log("Transactions loading time: \(startTime.timeIntervalSince)")
    return transactions
  }
}

// Composition

let loader = RemoteTransactionsRepository()
let loggingLoadTransactions = decorateWithLogging(loadTransactions: loader.loadTransactions)
let viewModel = TransactionsListViewModel(loadTransactions: loggingLoadTransactions)

```

Regardless if the implementation is more object-oriented or functional, the decorator-based design has two significant benefits:
1. complete separation of the logging as a cross-cutting concern from the feature code
1. the feature code has no awareness or dependencies on logging-related components

<div class="ml-embedded embedded-email-form" data-form="zRlsTe"></div>

Summary
-------

The decorator-based design gives us increased safety and flexibility.

The increased safety comes from the ability to change the logging behavior without changing the feature components. Notice in the code snippets above how the view model behavior did not change between having no logging and after adding logging with a decorator.

The increased flexibility allows us to move the feature code in a separate framework without dragging any logging dependency with it. We can also have different logging behavior with different decorators for different contexts (e.g., tests).

References
----------
* [https://www.essentialdeveloper.com/](https://www.essentialdeveloper.com/){:target="_blank"}
* [Dependency Injection Principles, Practices, and Patterns by Steven van Deursen and Mark Seemann](https://www.manning.com/books/dependency-injection-principles-practices-patterns){:target="_blank"}
* [https://en.wikipedia.org/wiki/Decorator_pattern](https://en.wikipedia.org/wiki/Decorator_pattern){:target="_blank"}
