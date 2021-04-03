---
layout: post
title:  "[swift] Rounded border button"
categories: ios swift uibutton
redirect_from:
  - /ios/swift/uibutton/2017/03/30/rounded-border-button/
---

Buttons can sometimes be difficult to distinguish from labels in the iOS interfaces with a lot of text. One way to make a button more noticeable is to add a border with a rounded corner to a UIButton view.

<img src="{{ site.url }}/assets/RoundedBorderButtonDemo.gif" style="width:100%;max-width:720px" title="Rounded Border Buttons">

Here is a complete RoundedBorderButton class you can use to achieve the same effect:

<script src="https://gist.github.com/srstanic/9637ef9a57d34546a9d7056082e8d45e.js"></script>

The class is using [@IBDesignable and @IBInspectable](http://nshipster.com/ibinspectable-ibdesignable/){:target="_blank"}<!-- markup clean_ --> attributes to enable you to modify the border properties in the Interface Builder and see the results immediately without running the app.

<img src="/assets/rounded-border-button-ib-properties.png" style="width:100%;max-width:720px" title="Rounded Border Button IB Properties">

RoundedBorderButton works best when the button type is set to Custom.
