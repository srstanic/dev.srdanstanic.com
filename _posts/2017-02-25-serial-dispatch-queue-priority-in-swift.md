---
layout: post
title:  "[swift] How to create a serial queue with specified priority?"
categories: ios swift gcd
redirect_from:
  - /ios/swift/gcd/2017/02/25/serial-dispatch-queue-priority/
---

To create a serial queue, normally you would write something simple like this:

`let queue = DispatchQueue(label: “com.mydomain.myqueue")`

If you wanted to define a priority for that queue, you could have done that by [retrieving a global queue of the required priority and setting it as a target](http://stackoverflow.com/a/17690878/517865){:target="_blank"}<!-- markup clean_ -->. But that’s the old way to do it and [it doesn’t work in Swift 3](https://bugs.swift.org/browse/SR-1859){:target="_blank"}<!-- markup clean_ -->.
Since Swift 3, once a dispatch queue is activated, it cannot be mutated anymore. Setting a target on an activated queue will compile but then throw an error in run time.

Fortunately, `DispatchQueue` initializer accepts other arguments besides `label` and one of them is `qos`.

`let queue = DispatchQueue(label: "com.mydomain.myqueue", qos: .background)`

Although [Apple documentation](https://developer.apple.com/reference/dispatch/dispatchqueue){:target="_blank"}<!-- markup clean_ --> states this initializer is available on iOS 10+, I’ve tested it and it also works on iOS 9.

If for whatever reason, you still need to set the target on an already created queue, you can do that by using the `initiallyInactive` attribute available since iOS 10:

`let queue = DispatchQueue(label: “com.mydomain.myqueue”, attributes: [.initiallyInactive])`

That will allow you to modify it until you activate it.

References
----------
* [https://devstreaming-cdn.apple.com/videos/wwdc/2016/720w6g8t9zhd23va0ai/720/720_concurrent_programming_with_gcd_in_swift_3.pdf](https://devstreaming-cdn.apple.com/videos/wwdc/2016/720w6g8t9zhd23va0ai/720/720_concurrent_programming_with_gcd_in_swift_3.pdf){:target="_blank"}
* [https://developer.apple.com/videos/play/wwdc2016/720/](https://developer.apple.com/videos/play/wwdc2016/720/){:target="_blank"}
