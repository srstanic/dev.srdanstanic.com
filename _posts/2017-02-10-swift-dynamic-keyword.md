---
layout: post
title:  "[swift] Dynamic keyword"
categories: ios swift
redirect_from:
  - /ios/swift/2017/02/10/swift-dynamic-keyword/
---


`dynamic` is a declaration modifier that you can apply to either function or variable declarations. It can only be used within a class and it tells the compiler to use dynamic dispatch over static dispatch.


Dynamic vs static dispatch
--------------------------

As Swift allows a class to override methods and properties declared in its superclasses, this means that the program has to determine at run time which method or property is being referenced and then perform an indirect call or indirect access. This technique is called dynamic dispatch and is applicable to types that support inheritance - classes. Value types like structs and enums cannot be extended through inheritance so they use static dispatch only.


Dispatch types in Swift
-----------------------

Swift actually supports three types of dispatch, ordered here from faster to slower:

1. direct dispatch aka static dispatch
2. table dispatch aka virtual dispatch
3. message dispatch

With a **direct dispatch**, there is a direct reference to the memory location where the method code is stored and that reference never changes in the run time. In the following example:

```swift
final class Point {
  func draw() {
    // draw implementation
  }
}
//...
let p = Point()
p.draw()
```

`Point` is a class that doesn't inherit from any other class and it's final so no other class can inherit from it. Compiler knows that there is only one implementation of the `draw()` method. Whenever `draw()` is called on a `Point` instance, compiler always references the same memory location where the code of that method is stored.

Now, let's say that we have a generic `Shape` class and two classes that represent specific implementations of it, `Line` and `Circle`:

```swift
class Shape {
    func draw() {
        // default draw implementation
    }
}

class Line: Shape {
    override func draw() {
        // line implementation
    }
}

class Circle: Shape {
    override func draw() {
        // circle implementation
    }
}

let shapes = fetchShapesFromAPI()
for shape in shapes {
    shape.draw()
}
```

In this example, direct dispatch can't be used when `draw()` is called. The compiler doesn't know in advance which of the three `draw()` implementations to point to because `shape` may be of different type in each iteration. So we need to use one of the other two dispatch types.

With a **table dispatch**, there is a lookup table of method pointers for each class in the class hierarchy - `Shape`, `Line` and `Circle`. Each table contains pointers for all methods in that class, including those that a class inherits from the parent classes. At the point of a method call, the program goes to the lookup table for the given class and finds the method pointer in it. That happens in run time.

Now, let's add `redraw()` method to the `Shape` class, which has some imaginary clean-up code and then calls `draw()` again.

```swift
class Shape {
    func draw() {
        // line implementation
    }
    func redraw() {
        // clean up code
        self.draw()
    }
}
...
for shape in shapes {
    shape.redraw()
}
```

That method is inherited but not overridden in the `Line` and `Circle` classes. With table dispatch, the lookup table for each of the three classes, `Shape`, `Line` and `Circle`, will have a copy of the reference to the `redraw()` method from the `Shape` class.

But with a **message dispatch**, there are no lookup tables. When the `redraw()` method is invoked, the program starts from the given class and then iterates the class hierarchy to find which class has the specified method implemented. For example, if an instance of the `Line` is in the array, the program will check if there is a `redraw()` method in the `Line` class - since there isn't one, it will move up to the `Shape` class and find it there.

Now, before we discuss the dynamic dispatch, let's briefly mention the compiler optimizations first.

Compiler optimizations
----------------------

[Swift compiler](https://www.swift.org/swift-compiler/){:target="_blank"}<!-- markup clean_ --> translates code from a human-friendly form into a machine-friendly form and in the process tries to optimize it to make it run faster. In the context of method dispatch, it will favor faster types over the slower. If the compiler can be sure at compile time that a particular method is going to be referenced, he might devirtualize it or even inline it.

**[Devirtualization](https://blogs.unity3d.com/2016/07/26/il2cpp-optimizations-devirtualization/){:target="_blank"}<!-- markup clean_ -->** is a common compiler optimization tactic which changes a virtual method call into a direct method call.

**[Inlining](http://compileroptimizations.com/category/function_inlining.htm){:target="_blank"}<!-- markup clean_ -->** is a compiler optimization tactic which replaces a direct method call with the method code inline.


Great, now will you finally tell me about dynamic dispatch?
-----------------------------------------------------------

Well, you will see both virtual dispatch and message dispatch referred to as dynamic dispatch. And although they are different mechanisms, they both are dynamic. But besides them being different mechanisms with different strengths and weaknesses, there is one other important distinction. Table dispatch is a part of pure Swift, while message dispatch is only supported in the Cocoa environments. The [Objective-C runtime library](https://developer.apple.com/documentation/objectivec/objective-c_runtime){:target="_blank"}<!-- markup clean_ --> is actually the one providing the message dispatch mechanism.

That finally brings us to the strength of the message dispatch. Since there is only one reference to one given method or property it can be safely and easily modified at run time. And Objective-C runtime provides tools to do it. These tools are the basis of cool dynamic features like KVC, KVO and UIAppearance.

Where does `dynamic` keyword come into play?
--------------------------------------------

Pure Swift classes don't normally use message dispatch and while classes derived from NSObject could use it, Swift compiler may optimize the way their properties and methods are referenced, which may break beforementioned dynamic features. If you want to use any of these dynamic features, you need to use `dynamic` modifier to enforce the message dispatch on that property or method. It can be used on both Swift and NSObject classes.

`dynamic` also implicitly adds the `objc` attribute and makes the declared property or method available in the Objective-C part of the program. To use the `dynamic` modifier, you must import Foundation, as this includes NSObject and the core of the Objective-C runtime.

Anything else?
--------------

`dynamic` can also be used on methods declared in class extensions to allow overriding them.

Let's modify the previous example:

```swift
class Shape {
    func draw() {
        // default draw implementation
    }
}

extension Shape {
    func redraw() {
        // clean up code
        self.draw()
    }
}

class Line: Shape {
    override func draw() {
        // line implementation
    }
    override func redraw() { // Compiler error: Declarations from extensions cannot be overridden yet
        //
    }
}
```

We have moved the `redraw()` method to the class extension. But now, if we want to override it in the subclass, the compiler will show an error. We can work around that by adding `dynamic` keyword to the `redraw()` declaration in the extension.

```swift
//...
extension Shape {
    dynamic func redraw() {
        // clean up code
        self.draw()
    }
}
//...
```

How does that make sense? Extension methods use static dispatch, so they can't be overridden. By adding `dynamic` to their declaration we force them to use message dispatch which allows overriding.


_Thanks to [Srđan Rašić](https://github.com/srdanrasic){:target="_blank"}<!-- markup clean_ --> for reviewing this post._


References
----------
* [https://krakendev.io/blog/hipster-swift](https://krakendev.io/blog/hipster-swift){:target="_blank"}
* [https://cocoacasts.com/what-does-the-dynamic-keyword-mean-in-swift-3/](https://cocoacasts.com/what-does-the-dynamic-keyword-mean-in-swift-3/){:target="_blank"}
* [https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations#Declaration-Modifiers](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations#Declaration-Modifiers){:target="_blank"}
* [https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151207/001707.html](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151207/001707.html){:target="_blank"}
* [http://cocoasamurai.blogspot.hr/2010/01/understanding-objective-c-runtime.html](http://cocoasamurai.blogspot.hr/2010/01/understanding-objective-c-runtime.html){:target="_blank"}
* [https://blogs.unity3d.com/2016/07/26/il2cpp-optimizations-devirtualization/](https://blogs.unity3d.com/2016/07/26/il2cpp-optimizations-devirtualization/){:target="_blank"}
* [https://en.wikipedia.org/wiki/LLVM#Intermediate_representation](https://en.wikipedia.org/wiki/LLVM#Intermediate_representation){:target="_blank"}
* [https://www.quora.com/Swift-can-be-compiled-with-the-LLVM-which-generates-Objective-C-code-out-of-it-and-then-Objective-C-will-be-compiled-to-machine-code-or-how-does-it-work/answer/Andrea-Ferro](https://www.quora.com/Swift-can-be-compiled-with-the-LLVM-which-generates-Objective-C-code-out-of-it-and-then-Objective-C-will-be-compiled-to-machine-code-or-how-does-it-work/answer/Andrea-Ferro){:target="_blank"}
* [https://www.infoq.com/articles/swift-objc-runtime-programming](https://www.infoq.com/articles/swift-objc-runtime-programming){:target="_blank"}
* [https://developer.apple.com/swift/blog/?id=27](https://developer.apple.com/swift/blog/?id=27){:target="_blank"}
