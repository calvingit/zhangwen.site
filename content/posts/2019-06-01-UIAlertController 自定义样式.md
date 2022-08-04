---
title: UIAlertController 自定义样式
slug: custom-UIAlertController
date: 2019-06-01
categories:
  - iOS
tags:
  - UIKit
toc: true
---

## 修改 UITextField 高度

不能直接改 frame.size.height，这样无效，需要添加约束：

swift:

```swift
alertController.addTextField { textField in
    let heightConstraint = NSLayoutConstraint(item: textField, attribute: .height, relatedBy: .equal, toItem: nil, attribute: .notAnAttribute, multiplier: 1, constant: 100)
    textField.addConstraint(heightConstraint)
}
```

objective-c:

```objc
NSLayoutConstraint * heightConstraint = [NSLayoutConstraint constraintWithItem:textField attribute:NSLayoutAttributeHeight relatedBy:NSLayoutRelationEqual toItem:nil attribute:NSLayoutAttributeNotAnAttribute multiplier:1 constant:32];
[textField addConstraint: heightConstraint];
```
