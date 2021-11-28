---
layout: post
title:  "[swift] Protocol types and naming"
categories: ios swift naming protocols
redirect_from:
  - /2021/03/24/naming-swift-protocols/
---

[John Sundell](https://www.swiftbysundell.com/about/){:target="_blank"}<!-- markup clean_ --> recently wrote an article on the topic of [categorizing and naming protocols in Swift](https://www.swiftbysundell.com/articles/different-categories-of-swift-protocols/){:target="_blank"}<!-- markup clean_ -->. As I was reading it, I realized I had a different take on several ideas shared in the article.

But before I go into the details, I'd like to say that I'm a big fan of John's work. The first thing I saw from John was his conference talk on [system design](https://www.swiftbysundell.com/videos/the-lost-art-of-system-design/){:target="_blank"}<!-- markup clean_ -->. Afterward, I started listening to his [podcast](https://www.swiftbysundell.com/podcast/){:target="_blank"}<!-- markup clean_ --> and reading his [articles](https://www.swiftbysundell.com/articles/){:target="_blank"}<!-- markup clean_ --> regularly. I appreciate his efforts to educate the audience about the best practices of programming in Swift. And this post is not about whether he's wrong and I'm right, or vice versa. This post is about having different perspectives on the categorization of components and naming them. And the difference in opinion on these topics can create a lot of friction in teams. There isn't always going to be a right and wrong approach to a certain aspect of programming, but there can always be a shared convention within a team working on the same project. And it's important to have such conventions and consistency in following them because it helps the readability of the codebase. Btw. does anyone still choose tabs over spaces? 🙊

In this particular article, as always, John brings some good points about writing generic code with protocol-oriented design and hiding details of third-party interfaces behind internal protocols. As for the protocol categorization, he suggests four categories of protocols:
1. Action enablers
2. Requirement definitions
3. Type conversion
4. Abstract interfaces

While it's a reasonable categorization, I would prefer one that provides more straightforward guidelines on the protocols' naming.

Protocol categorization and naming
----------------------------------

If we look at the [Swift API design guidelines](https://swift.org/documentation/api-design-guidelines/){:target="_blank"}<!-- markup clean_ -->, there are two bullets describing the naming of protocols.

> - Protocols that describe *what something is* should read as nouns (e.g. `Collection`).
> - Protocols that describe a *capability* should be named using the suffixes `able`, `ible`, or `ing` (e.g. `Equatable`, `ProgressReporting`).

We can take the *capability* category, and we can divide it into two subcategories, one where the implementer is a subject performing an action, and the other where the implementer is an object the action is performed on.

In summary, we can define three categories of protocols:
1. the implementer *is something* - named with a noun
2. the implementer *is doing something* - named with an adjective ending with `ing`
3. *something is done to* the implementer - named with an adjective ending with `able` or `ible`

I haven't yet found short and meaningful names for these categories, but please let me know in the comments if you have some suggestions.

Examples
--------

Now let's go through some of the examples John provided.

There is a `Loadable` protocol defined which unifies several types that load various objects or values.

```swift
protocol Loadable {
    associatedtype Result
    func load() throws -> Result
}
```

If we look at what this type does, it performs an action of loading. It is not being loaded itself. The `Result` is the thing being loaded. Thus an appropriate name for this protocol would be `Loading` or `ResultLoading`.

```swift
protocol Loading {
    associatedtype Result
    func load() throws -> Result
}
```

And if we want the result to be `Loadable`, we can define the protocol that way and make the result type conform to it.

```swift
protocol Loadable {
    func load() throws -> Self
}

extension SomeLoadableType: Loadable {
    func load() throws -> Self { ... }
}
```

But that's probably not a good idea. A loading operation usually requires some dependencies to a database or a network interface, and the result of a loading operation is typically a data object. And data objects should only contain data and not hold references to the infrastructure components.

The next example discussed is `Cachable`.

```swift
protocol Cachable: Codable {
    var cacheKey: String { get }
}
```

John argues that this is not a good name because the `Cachable` protocol doesn't contain any methods that would perform the caching. But the question is, why should `Cachable` have any methods to perform caching? The protocol describes something that can be cached and the requirement for that something is to provide a `cacheKey`. `Cachable` is a suitable name for it. If this type itself needed to perform caching actions, then a more fitting name for it would be `Caching`.

The following example is a more complicated one - the `Swift.Collection.Sequence` protocol from the Swift Standard Library.

```swift
public protocol Sequence {
    associatedtype Iterator: IteratorProtocol
    func makeIterator() -> Self.Iterator
}
```

First, let's look at the `IteratorProtocol` protocol. Adding a `Protocol` suffix to the protocol is easy and convenient, but I have yet to encounter an example where that was the best option. I've found that picking from the three categories defined above is a better approach. So how can we improve the naming here?

Whenever I deal with generics, I like to suffix the type parameter name with a `Type` suffix. That makes it very easy to distinguish between actual types and their placeholders in the generic code, and makes the code more readable. So in the `Sequence` example, I would propose the following naming.

```swift
public protocol Sequence {
    associatedtype IteratorType: Iterator
    func makeIterator() -> Self.IteratorType
}
```

And following the same convention, let's update the earlier `Loading` protocol.

```swift
protocol Loading {
    associatedtype ResultType
    func load() throws -> ResultType
}
```

But getting back to the `Sequence` example, naming the `Iterator` as such doesn't sound quite right, because the `Iterator` doesn't fit the *is something* category. Apple's documentation summarizes the `Iterator` type like this:

> A type that supplies the values of a sequence one at a time.

The `Iterator` has a `next()` method which provides the next item in the sequence. The implementer of this protocol is something that can be iterated. So a more appropriate name for the protocol would be `Iterable`. And finally the `Sequence` protocol would look like the following.

```swift
public protocol Sequence {
    associatedtype IterableType: Iterable
    func makeIterable() -> Self.IterableType
}
```

The next example is a `ColorProvider` protocol described as an interface for a container of colors.

```swift
protocol ColorProvider {
    var foregroundColor: UIColor { get }
    var backgroundColor: UIColor { get }
}
```

Again, I find using a noun here inappropriate because this type is not in the *is something* category, but in the *is doing something* category. This type provides colors, so it should be named `ColorProviding`. There are many examples of type names like `Manager`, `Coordinator`, and `Generator`, where a verb is turned into a noun. When used in protocol names, it would be more fitting to turn these verbs into adjectives - `Managing`, `Coordinating`, `Generating`.

Another example from John's article is a protocol that describes something that has a `title` property. This one is tricky. John placed it into the *type conversion* category suggesting the `TitleConvertible` name.

```swift
protocol TitleConvertible {
    var title: String { get }
}
```

I get the line of thinking there, but since the title is not a type to which the implementer can be converted, I would look further for a good candidate for this protocol’s name.   And I would even say that the fact that something has a title doesn't justify having a separate protocol. To put it differently, it's not useful for many different types to conform to the same generic `title`-requiring protocol, because it's unlikely that there is going to be generic logic applicable to anything that has a title. But if there is something that *is something* and also has a title, it might be worth having a protocol for it. For example, a view that has a title could be defined as a `TitledView` protocol, which extends from a `View` protocol.

```swift
protocol TitledView: View {
    var title: { get set }
}
```

Conclusion
----------

To reiterate the previous points, I would suggest using the following categorization and naming guidelines for the protocols in Swift:
1. If the implementer *is something*, name the protocol with a noun, e.g. `Sequence`, `View`, `Repository`
2. If the implementer *is doing something*, name the protocol with an adjective ending with `ing`, e.g. `Loading`, `Generating`, `Coordinating`
3. If *something is done to* the implementer, name the protocol with an adjective ending with `able` or `ible`, e.g. `Comparable`, `Codable`, `Cachable`

If you have any thoughts, remarks, or a different viewpoint on any of the above, please let me know in the comments.