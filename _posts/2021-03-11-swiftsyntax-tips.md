---
layout: post
title:  "[swift] Tips for parsing Swift code with SwiftSyntax"
categories: ios swift
redirect_from:
  - /2021/03/11/swiftsyntax-tips/
---

I recently worked on a project with a goal to consolidate and standardize localization efforts in a sizeable iOS application written in Swift. This app uses a 3rd party localization service that supports Over-The-Air (OTA) localization. In short, it means that the live app can fetch the latest localizations from the server and there is no need to resubmit the app to the App Store to update the localizations. As a result of this feature, there are no `.strings` files in the project. All the localization keys and localized values are stored in a proprietary formatted database file used by the 3rd party localization service SDK.

The app has been in development for over 6 years, and presumably, many keys present in the database weren’t used anymore. Having a growing number of unused keys lingering in the database increases the localization costs when entering new markets. So one of the goals was to clean up the unused keys. And the challenge was to figure out which localization keys are currently being used in the app. The only way to know for sure is to find all localization keys referenced in the code. And given that the project has been in development for so long, there was a handful of different patterns for referencing localization keys.

Here is where SwiftSyntax enters the stage. I needed a reliable Swift code parsing tool to collect these localization keys and SwiftSyntax emerged from the research as the best tool for the job.

What is SwiftSyntax and how to use it?
--------------------------------------

 [SwiftSyntax](https://github.com/apple/swift-syntax){:target="_blank"}<!-- markup clean_ --> is a library for parsing, inspecting, generating, and transforming Swift source code. If you are not familiar with it, the basics of the tool are pretty well covered in these two articles:
* [Swift​Syntax](https://nshipster.com/swiftsyntax/){:target="_blank"}<!-- markup clean_ --> by [Mattt](https://twitter.com/mattt){:target="_blank"}<!-- markup clean_ -->
* [An overview of SwiftSyntax](https://medium.com/@lucianoalmeida1/an-overview-of-swiftsyntax-cf1ae6d53494){:target="_blank"}<!-- markup clean_ --> by Luciano Almeida

To explain how SwiftSyntax works in a single sentence, it builds an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree){:target="_blank"}<!-- markup clean_ --> of the Swift source code and then traverses that tree while letting you react whenever it encounters a type of tree node which interests you.

And to provide an example, let's take a `UIViewController` that shows a localized string in its root view.

```swift
class ViewController: UIViewController {
    @IBOutlet private weak var label: UILabel!

    override func viewDidLoad() {
        super.viewDidLoad()
        label.text = NSLocalizedString("Hello World!", comment: "")
    }
}
```

The goal here is to collect the `Hello World!` string from this code. We can achieve that by implementing a subclass of the `SyntaxVisitor` class and override one of its `visit(_:)` methods. The `SyntaxVisitor` class is the component that walks the syntax tree and the `visit(_:)` methods provide the hook for your code to run whenever a certain type of node is encountered. The `NSLocalizedStringKeyCollector` subclass specifically overrides an overload of the `visit(_:)` method which accepts a `FunctionCallExprSyntax` node as an argument.

```swift
final class NSLocalizedStringKeyCollector: SyntaxVisitor {
    var result: [String] = []

    override func visit(_ node: FunctionCallExprSyntax) -> SyntaxVisitorContinueKind {
        guard
            let identifierExpr = node.calledExpression.as(IdentifierExprSyntax.self),
            identifierExpr.identifier.text == "NSLocalizedString",
            let firstArgumentExpr = node.argumentList.first?.expression.as(StringLiteralExprSyntax.self),
            let localizationKey = firstArgumentExpr.segments.first?.as(StringSegmentSyntax.self)?.description
        else {
            return .visitChildren
        }
        self.result.append(localizationKey)
        return .skipChildren
    }
}
```

We can use the `NSLocalizedStringKeyCollector` class to walk the syntax tree and collect any localization keys specified as string literals passed directly into the `NSLocalizedString` function.

```swift
let code = readCodeAsStringFromFile() // reads our ViewController code from a swift file
let collector = NSLocalizedStringKeyCollector()
if let syntaxTree = try? SyntaxParser.parse(source: code) {
    collector.walk(syntaxTree)
    print(collector.result) // prints ["Hello World!"]
}
```

Tip 1: `Syntax.syntaxNodeType` will help you understand the tree structure
--------------------------------------------------------------------------------

The implementation of the `NSLocalizedStringKeyCollector` goes into the children of the `FunctionCallExprSyntax` node looking for a specific structure of nodes. To understand the different types of nodes your code is built from, you will either use autocomplete to explore individual node's properties, or click-through into the types themselves to explore them. The challenge here is that the syntax node's properties will often have a generic `Syntax` or `ExprSyntax` type, and you will need to know what is the specific node type of the property value. There are many types of nodes and unless you are already very familiar with the full set, it's going to be hard to guess which specific type you are dealing with.

```swift
...
guard
    let identifierExpr = node.calledExpression.as(IdentifierExprSyntax.self),
    identifierExpr.identifier.text == "NSLocalizedString",
    let firstArgumentExpr = node.argumentList.first?.expression.as(StringLiteralExprSyntax.self),
    let localizationKey = firstArgumentExpr.segments.first?.as(StringSegmentSyntax.self)?.description
else {
...
```

As it's visible from this snippet, to dig deeper into the tree structure, there are several steps where we need to know the specific node type we want to "cast" into with the `.as()` method. And this is where the `syntaxNodeType` property comes in handy. An easy way to find out the type of a node is to print the value of the `syntaxNodeType` property to the console. But digging through the tree structure node by node can be tedious. Luckily, there is a really cool tool, [Swift Abstract Syntax Tree visualizer](https://swift-ast-explorer.com/){:target="_blank"}<!-- markup clean_ -->, which automates this process completely.

Tip 2: Generalize the tree walking logic
----------------------------------------

I had a handful of syntax visitors and each of them had a purpose of finding some kind of a result, either a string or a certain node type. For each of these visitors, I needed to implement a method that would reset their result, walk the tree, and return the result. After creating a couple of these, I decided to create a generic `SyntaxNodeProcessor` component that extracts the common logic and provides a unified interface.

```swift
class SyntaxNodeProcessor<Result, SyntaxNodeType: SyntaxProtocol>: SyntaxVisitor {
    var result: Result?

    func resetResult() {
        result = nil
    }

    func process(_ node: SyntaxNodeType) -> Result? {
        resetResult()
        walk(node)
        return result
    }
}
```

The usage of this component doesn't decrease the amount of code significantly, but it provides consistency and reduces the cognitive load when writing new visitor types. If you have a number of syntax visitors in your code, you might find `SyntaxNodeProcessor` class useful as well.

The new implementation of the `NSLocalizedStringKeyCollector` based on the `SyntaxNodeProcessor` would look like the following.

```swift
final class NSLocalizedStringKeyCollector: SyntaxNodeProcessor<[String], SourceFileSyntax> {
    override func resetResult() {
        result = []
    }

    override func visit(_ node: FunctionCallExprSyntax) -> SyntaxVisitorContinueKind {
        guard
            let identifierExpr = node.calledExpression.as(IdentifierExprSyntax.self),
            identifierExpr.identifier.text == "NSLocalizedString",
            let firstArgumentExpr = node.argumentList.first?.expression.as(StringLiteralExprSyntax.self),
            let localizationKey = firstArgumentExpr.segments.first?.as(StringSegmentSyntax.self)?.description
        else {
            return .visitChildren
        }
        self.result.append(localizationKey)
        return .skipChildren
    }
}
```

And here is how the call site looks when the instance of the reimplemented `NSLocalizedStringKeyCollector` is used in a loop.

```swift
let collector = NSLocalizedStringKeyCollector()
for file in files {
    let code = readCodeAsString(from: file)
    if let syntaxTree = try? SyntaxParser.parse(source: code) {
        print(collector.process(syntaxTree))
    }
}
```

Conclusion
-------
Parsing Swift code with SwiftSyntax turned out to be much easier than I expected. If you need to parse Swift code, you should consider SwiftSyntax. And if you end up using it, I hope you find the tips in this post useful.



