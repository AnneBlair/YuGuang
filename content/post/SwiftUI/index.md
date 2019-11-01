---
title: ' SwiftUI 使用 UIKit 作为UI控件不足的补充'
subtitle: '' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2018-07-14T00:00:00Z"
lastmod: "2018-07-14T00:00:00Z"
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Placement options: 1 = Full column width, 2 = Out-set, 3 = Screen-width
# Focal point options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
image:
  placement: 2
  caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/CpkOjOcXdUY)'
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

# SwiftUI 使用 UIKit 作为UI控件不足的补充
本文主要目的就是解决目前 **SwiftUI** 中 **TextView** 的不足.
`SwiftUI` 在 `Xcode` **Version 11.1** 中的 `TextView` 依然有许多问题, 文本在视图内部无法正确显示是使用的当务之急，目前的解决办法是利用 `UITextView`进行  **“ 曲线救国 ”**

## 先了解Representable 协议

`Representable` 协议允许我们在 **Swift** 中呈现 `UIView`,  `NSView`, `WKInterfaceObject`

 **其关系对应如下：**
 
| UIView | UIViewRepresentable |
| --- | --- |
| NSView | NSViewRepresentable |
| WKInterfaceObject | WKInterfaceObjectRepresentable |`

-------
### 一. Representable 协议有两个必须执行的方法
* **1. Make Coordinator(可选)**
		初始化过程中首先调用
		
* **2. makeUIView(必须)**
		是创建你希望在SwiftUI中呈现的视图或者控制器的地方
		
* **3. updateUIView(必须)**
		该方法是把这个视图，更新到当前配置的地方
		
* **4. Dismantle(可选)**
		视图消失之前调用

**在初始化过程中首先调用 `Make` 方法，然后再调用 `update`方法， `update`方法可以多次调用，只要是请求更新的时候都可以调用， 这两个方法是呈现视图仅有的两个必须方法**



-------


### 二. UIViewRepresentableContext
上面说的两个视图呈现的必须方法，在实际的项目中使用不可能是简单的视图展示，更多的是动作事件或者委托，比如：
1. 在SwiftUI 中读取内容，并作出响应
2. 了解当前视图上是否有动画
3. 等等...

为了更好的处理**视图**与**SwiftUI**直接的关系，**Representable**协议是关键的存在；它可以在视图、视图控制器、以及界面控制器中使用。
#### **Representable Context** 有三个属性:

* **1.Coordinator** 
用于实现常见模式，帮助你协调你的视图和 **SwiftUI**
包括委派，数据源和目标动作.

* **2.Environment**
帮助你读取**SwiftUI**的环境, 这可能是系统环境，例如配色方案或尺寸类别或设备方向. 也可以是应用app自定义的环境属性.

* **3.Transaction**
让我们的视图知道**SwiftUI**中是否有动画, 可表示的上下文可用于视图，视图控制器以及接口控制器.

-------

### 三. UIViewRepresentableContext 的使用
案例：在 **SwiftUI** 视图中嵌入基于 **UIKit** 的 **UITextView**的控件

#### 上面我们回顾: 

-------

```
一.Representable 协议有两个必须执行的方法

1. Make Coordinator(可选)
2. Make View
3. Update View
4. Dismantle View(可选)

二. Representable Context

1. Coordinator
2. Environment
3. Transaction
	
```

在 `Make View` 和 `Update View` 中我们传入 `Representable Context`参数，由于 `Representable Context` 的三个属性: `Coordinator` `Environment` `Transaction`

如果要使用 `Coordinator`，则需要在可选的 `Make Coordinator` 方法中创建，在初始化过程中首先调用的是 `Make Coordinator`方法, 我们把 **UIKit**相关的动作事件或者委托在 `Make Coordinator`中处理好,这样关于 **UIKit** 相关的 **动作事件或者委托**的就可以传递到 SwiftUI 里面了;

**我们看例子：**

```
struct TextView: UIViewRepresentable {
    
    class Coordinator : NSObject, UITextViewDelegate {
        var textView: TextView

        init(_ uiTextView: TextView) {
            self.textView = uiTextView
        }

        func textView(_ textView: UITextView, shouldChangeTextIn range: NSRange, replacementText text: String) -> Bool {
            return true
        }

        func textViewDidChange(_ textView: UITextView) {
            print("text now: \(String(describing: textView.text!))")
            self.textView.text = textView.text
        }
    }
    
    @Binding var text: String

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    func makeUIView(context: Context) -> UITextView {
        let textView = UITextView()
        textView.delegate = context.coordinator
        textView.isScrollEnabled = true
        textView.isEditable = true
        textView.isUserInteractionEnabled = true
        return textView
    }

    func updateUIView(_ uiView: UITextView, context: Context) {
        uiView.text = text
    }
}
```

1.先调用 `makeCoordinator` 创建 `UIViewRepresentableContext`用于处理 `UIKit` 中的动作事件或者委托；
2. `makeUIView` 中添加动作事件或者委托的目标  `textView.delegate = context.coordinator`

### 四. 最后
关于如何在SwiftUI中打造使用 UITextView上面已经很详细，更多的自定义处理需要根据需求进一个扩展；但是核心点就是两点的使用：

**1. UIViewRepresentable**
**2. UIViewRepresentableContext**

-------


本文在苹果开发者官网教程中有 **SwiftUI** 直接使用 **UIKit** 英文[教程](https://developer.apple.com/tutorials/swiftui/interfacing-with-uikit)


