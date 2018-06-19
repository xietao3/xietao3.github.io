---
title: WWDC 2017：高级开发应该掌握的自动布局技巧
layout: post
author: 谢涛
date: '2018-06-14 10:40:11 +0800'
categories: iOS
---

> 构建 app 时使用的自动布局技术，其实就是建立视图与视图之间关系。而约束是建立视图间关系的纽带，帮助我们的 app 可以适应各种尺寸的屏幕，在应对花样百出的布局需求时游刃有余。
><br><br>本文已收录至 [iOS 成长之路3期·WWDC17内参](https://xiaozhuanlan.com/wwdc17)

## 前言

如果你以前从未使用过``Autolayout``，现在网上已经有很多很优秀的教程，包括往届 WWDC 中 sessions 视频资源都可供查看学习。在本文中将不再重复基本的使用方法，更多的去介绍一些更加复杂的场景中的应用，本文中技术结合实例使你更容易理解吸收。让我们一起来看看与``Autolayout``相关的六种技术与应用，这些内容都非常实用，在日常开发中一定会经常使用到，相信本文一定不会让你失望。

## 1. 运行时变换布局（Changing layout at runtime）

通常我们不仅可以在 app 中使用约束来对视图进行简单的定位，也可以组合使用以达到更复杂的效果。我们今天要讲的第一种技术点，便在运行时改变布局。如下图，在我们界面的顶部，有一个滑块区域。现在我们需要一个将滑块视图上移并且最终隐藏的功能。

![顶部滑块区域](https://user-gold-cdn.xitu.io/2018/6/14/163fc346aafc0171?w=454&h=262&f=png&s=14668)

### 1.1 利用高度约束隐藏视图

通常我们希望约束在设置好之后不需要再次调整，尽量让结构清晰简单。现在我们来思考一下，从布局的角度使用最简单的方式实现这个功能，一般情况下，我们把这个区域视图高度缩短至0即可。但是如果我们真的添加上一个高度约束，并且设置为0。我们将在 Interface Builder 中发现一些警告。

![顶部滑块约束存在冲突](https://user-gold-cdn.xitu.io/2018/6/14/163fc34ee3e124dc?w=454&h=416&f=png&s=18507)

### 1.2 避免冲突

在图片中可以看到布局中的这些红线，这意味着我们设置的约束存在着一些冲突。之所以出现冲突，是因为我们设置的这些约束让布局引擎去做了一些不能同时并存的事情。而这个冲突出现是因为我们设置了高度为0的同时，无法保持足够的高度以满足该控件内部的内容显示。

![容器](https://user-gold-cdn.xitu.io/2018/6/14/163fc35457f0e844?w=454&h=188&f=png&s=10973)

为了解决这个问题，我们将 slider 和 label 所在的视图放进一个``warppingView``中，如图中橙色方框。在我们缩短``warppingView``的高度时，我们也要保证``warppingView``内部子视图的高度，并且满足子视图相关的约束，在启用 clips ToBounds 属性后，超出``warppingView``内部坐标系范围的内部控件在显示时将被裁剪掉。这样就达到了隐藏视图元素的效果，如下图效果，灰色区域将被裁剪不显示。

![隐藏滑块区域](https://user-gold-cdn.xitu.io/2018/6/14/163fc35a59800d2a?w=454&h=195&f=png&s=9889)

让我们来看看在 Xcode 中是如何做到的，我们需要在运行时控制``warppingView``的高度，所以我们将为``warppingView``手动创建一个高度约束``zeroHeightConstraint``，在运行时设置``zeroHeightConstraint``为0，并且在用户点击 Edit 按钮时，激活该约束。这样我们仍然会和之前一样出现冲突的情况，我们需要将滑块区域视图底部到``warppingView``底部边缘的约束禁用，避免了约束冲突，这样``warppingView``就可以正常缩短高度了。

![禁用底部约束](https://user-gold-cdn.xitu.io/2018/6/14/163fc3648d447ea7?w=454&h=234&f=png&s=11008)

### 1.3 实现代码

接下来看看完整代码，在我们控制器的子类中，我们持有3个属性：

* ``warppingView``：外部容器视图
* ``edgeConstraint``：底部边缘的约束 
* ``zeroHeightConstraint``：一个存储0高度约束的属性

```
@IBOutlet var warppingView: UIView!
@IBOutlet var edgeConstraint: NSLayoutConstraint!
var zeroHeightConstraint : NSLayoutConstraint!
```

我们创建了按钮点击事件，在响应按钮事件函数中，我们首先要保证``zeroHeightConstraint``已被创建。接着我们还希望这一个事件让视图可以在显示和隐藏间切换，所以我们要对一些约束做禁用和激活操作，做完这些就会得到我们想要的切换效果。

```
@IBAction func toggleDistanceControls(_ sender: Any) {
        if zeroHeightConstraint == nil {
            zeroHeightConstraint = warppingView.heightAnchor.constraint(equalToConstant: 0)
        }
        
        let shouldShow = !edgeConstraint.isActive
        
        if shouldShow {
            zeroHeightConstraint.isActive = false
            edgeConstraint.isActive = true
        }else{
            edgeConstraint.isActive = false
            zeroHeightConstraint.isActive = true
        }
    }
```

需要特别注意的是，**在激活一个约束前务必先禁用另外一个约束**。在这些简单的切换禁用和激活代码，遵守这一点让我们避免了冲突，如果约束中一旦存在冲突，控制台就会提醒我们：嘿，我检测到这些约束是互相冲突的😂。例如，我们激活了``zeroHeightConstraint``约束，而底部约束``edgeConstraint``还未被禁用，这个时候我们就会看到控制台打印出冲突信息。

### 1.4 加入动画

加入这些代码重新运行后你会发现我们的界面正确显示和隐藏了，但是我还想为这个过程加上动画，让用户可以看到视图切换过程能够提高用户体验。在这里我们使用``UIView animation block``来实现动画，``UIView animation``将捕捉并且动画化整个过程。

```
UIView.animate(withDuration: 0.25) {
    self.view.layoutIfNeeded()
}
```

这里得到的动画效果，也并不是我想要的最终效果，我们还需要做最后一点调整，但是这不需要修改我们的代码，我们只需要将底部边缘的约束:``edgeConstraint``属性更换成连接到顶部边缘的约束，改成一个底部对齐的效果。整个动画效果发生改变，我确认这就是我需要的最终效果。

![禁用顶部约束](https://user-gold-cdn.xitu.io/2018/6/14/163fc370a9baaa2c?w=454&h=462&f=png&s=20891)

具体效果可以查看我们的[Demo(非苹果官方)](https://github.com/xietao3/FindMyDates)，通过上面这些内容我们可以知道，怎样通过运行时改变约束来动态调整我们 app 中的布局。

![](https://user-gold-cdn.xitu.io/2018/6/14/163fc62c8f15ebbe?w=750&h=765&f=gif&s=1251779)

## 2. 跟踪触摸手势（Tracking touch）

现在我们来看看改变布局的另一种方法，**我保证它既简单又炫酷**。我们将用它来跟踪触摸手势。我们在我们下图的 app 的中央区域有一张卡片，我们希望卡片能随着触摸手势移动，随着靠近边缘的时候，会有一些旋转，一旦你的手离开屏幕，卡片就会弹回屏幕中间。

![卡片](https://user-gold-cdn.xitu.io/2018/6/14/163fc38a51d1b2de?w=1181&h=664&f=jpeg&s=39568)

### 2.1 frame 饮水知源

通常一个控件在屏幕上的位置由它的 frame 决定，而 frame 又源起何处呢？

* **Layout engine owns frame**。当我们使用``Autolayout``并使用约束控制此视图时，布局引擎将会持有此视图的 frame 。
  * **Value derived from constraints**。frame 的值是从这些约束中计算出来的。
* **transform property offsets from frame**。还有另一个属性会影响视图在屏幕上的位置，那就是 transform ，在 transform 属性源起于 frame 。
* **CGAffineTransform = translation + rotation + scale**。通过``CGAffineTransform``，它可以帮助我们为视图加入平移 ，旋转和缩放等变换，在从约束中计算出 frame 之后，将其应用在 transform 中。

### 2.2 加入监听手势

再回到需求上，如果我们想要中间的卡片随着我的手势移动，那我们就要加入一个手势识别器，并且拖线连接到代码中，添加监听手势的方法，在该方法中我们可以访问手势识别器的各种属性。此外还将我们要移动的卡片也通过拖线创建了属性。

```
@IBOutlet weak var cardView: UIImageView!
@IBAction func panCard(_ sender: UIPanGestureRecognizer) {}
```

### 2.3 加入位移和旋转

接下来我们要通过手势识别器监听用户手势移动，得到位移结果后转换成 transform 应用在``cardView``上。在这里有``transform``函数帮我们进行了位移和轻微的旋转，这个时候，卡片将会随着你的手指移动伴随着轻微的旋转。

```
func transform(for translation: CGPoint) -> CGAffineTransform {
    let moveBy = CGAffineTransform(translationX:translation.x, y: translation.y)
    let rotation = -sin(translation.x/(cardView.frame.width * 4.0))
    return moveBy.rotated(by: rotation)
}
```

### 2.4 位置还原

但是当我放开手指时，卡片停留在原位，没有回到屏幕中央，因为我们并没有去重置卡片的 transform 属性。当我再次触摸并移动，我们会看到它会回到原来的位置，这是因为我们开始了一个新的位移，新的位移关联的原来的 frame 。总之这不是我想要的效果，我希望在用户手指离开屏幕后，卡片能够立即回到屏幕中间的位置。我们可以通过手势识别器的状态来做到这一点。我们在``state``为``end``的时候，将重置 transform 并且加入弹簧动画。加入这部分代码运行 app ，在我松开卡片后它会弹回中间的位置。

```
@IBAction func panCard(_ sender: UIPanGestureRecognizer) {
    switch sender.state {
    case .changed:
        let translation = sender.translation(in: view)
        cardView.transform = transform(for: translation)
    case .ended:
        UIView.animate(withDuration: 0.5, delay: 0, usingSpringWithDamping: 0.4,
initialSpringVelocity: 1.0, options: [], animations: {
            self.cardView.transform = .identity
        }, completion: nil)
    default:
        break;
    }
}
```

简单的几行代码，实现了一个很有意思的交互效果，在这些内容里面，我们可以看到 frame 它不仅是通过约束来计算出，也会受到 transform 的影响，视图的 frame 中蕴含多种属性的组合效果。

![](https://user-gold-cdn.xitu.io/2018/6/14/163fc636501abadd?w=750&h=1334&f=gif&s=3274479)

## 3. 动态字体（Dynamic type）

``Dynamic type``是 iOS 中提供了一组文本样式，文本样式包含了标题、副标题、正文等样式，而且用户可以控制这些样式字体大小的技术。在 iPhone 的短信消息中，如果用户喜欢大一点的字体，通过在设置中进行设置后，我们将看到消息界面会有所变化，字体变大了，消息气泡和输入文本也变大了。日历和其他一些地方也具备类似的功能。

![改变字体前](https://user-gold-cdn.xitu.io/2018/6/14/163fc3963b6c08d9?w=1701&h=850&f=png&s=121305)

![改变字体后](https://user-gold-cdn.xitu.io/2018/6/14/163fc3990fba754c?w=1701&h=850&f=png&s=121721)

相信这个时候你一定会好奇，如何才能在我们自己的 app 实现这个功能呢？另外在调整字体大小时，如果不相应地调整我们的布局，容易造成视图重叠，对用户体验来说是非常不好的。幸运的是，``Autolayout``可以很轻松地帮我们搞定这个问题。

### 3.1 支持 Dynamic Type

所以赶紧让我们来看看是怎么实现的吧，打开 IB 界面，选中你要支持 Dynamic Type 的 label ，查看 label 的属性，勾选``automatically adjust font``，如果你眼睛够敏锐的话，你能看到上面出现了一个警告，原因是因为``automatically adjust font``属性生效，该属性要求 label 设置指定的文本样式。这里要将系统默认字体更换为``caption one``，该样式和默认字体12号大小相对应。

![支持Dynamic Yype](https://user-gold-cdn.xitu.io/2018/6/14/163fc3a5bbd05af0?w=340&h=353&f=png&s=33814)

### 3.2 通过 Accessibility Inspector 改变字体大小

设置好这些再重新运行，你会发现和之前并没有什么不同，这是因为我们还没有改变文本样式的大小，我们可以在设置中调整字体大小，但是这种方式需要来回切换不够直观，所有我们用另外一种方法，点击顶部导航条``Xcode``->``Open Develop Tool``->``Accessibility Inspector``->``target``切换至模拟器->选择设置标签，就可以看到修改字体大小的滑块了。这个时候滑动滑块就能看到我们的 label 字体在实时地改变。如果我们连接了 iPhone ，我们也可以将``target``切换至我们的 iPhone 。

![Accessibility Inspector](https://user-gold-cdn.xitu.io/2018/6/14/163fc3ac15cbee08?w=454&h=282&f=png&s=23590)

### 3.3 根据字体大小动态调整布局

在下图中可以看到，如果我们把字体调整到非常大的时候，我们的 label 就会发生重叠，接下来我们要解决这个问题。

![视图重叠](https://user-gold-cdn.xitu.io/2018/6/14/163fc3b0847812ec?w=454&h=842&f=png&s=59737)

首先我们创建了一个文本区域，这个文本区域会随着字体变大而增高，所以我们只要将底部 label 被限制在底部，顶部 label 被限制在顶部，再在两者之间添加了一个垂直间距约束，使两个标签始终保持足够垂直间距，避免使用固定高度约束，这样 label 的高度会随着字体变大而增高，接着 label 又会将文本区域给撑高。这个时候可以打开``Accessibility Inspector``来测试改变我们 app 的字体大小，你会发现我们的文本区域会随着字体的变化而改变高度。

![根据字体大小动态调整布局](https://user-gold-cdn.xitu.io/2018/6/14/163fc3e7508dbc4a?w=454&h=782&f=png&s=79656)

在这中间我们并不需要做很多处理，就能实现这样一个非常实用的功能，特别是当你有一些需要读者阅读文字的的需求 ，相信 Dynamic Type 能够帮助你，让你的 app 更加强大。

![](https://user-gold-cdn.xitu.io/2018/6/14/163fc63eeccd70a3?w=750&h=1334&f=gif&s=3523662)

## 4. 安全区（Safe area）

接下来要介绍的内容在之后你可能会频繁使用到，所以一定要搬好小板凳认真看。当你新建了一个控制器，控制器有一个导航条和一个底部标签栏，如何保证你的内容主体不被导航条和标签栏遮挡？可能你已经听说过在 iOS 11 上有了新的 layout guide ，称之为``Safe Area Layout Guide``。

![Safe Area](https://user-gold-cdn.xitu.io/2018/6/14/163fc3f07c3f8dba?w=1181&h=664&f=jpeg&s=20468)

### 4.1 Safe Area Layout Guide 更易使用

这是``UIView``的新特性，它适用于自动布局，它是夹在导航条和标签栏之间的一个矩形，在这个矩形区域中你可以放心地为你的视图添加约束。在这之前，你可能不得不使用``UIViewController``的``Top Layer Guide``和``Bottom Layer Guide``，现在在``Safe Area``中这些都已经通通被丢弃了。

![使用Safe Area](https://user-gold-cdn.xitu.io/2018/6/14/163fc3f38ee95791?w=1181&h=664&f=jpeg&s=29869)

``Safe Area Layout Guide``使用起来更加简单，也更容易理解，如同字面意思，可以安全的让你的视图安全地呆在导航条和标签栏中间，不被遮挡，不管是尺寸的变化和屏幕旋转，它都会自动做相应地调整。

![使用Safe Area 横屏](https://user-gold-cdn.xitu.io/2018/6/14/163fc3f76e82aa76?w=1181&h=664&f=jpeg&s=31387)

### 4.2 如何使用 Safe Area Layout Guide

``Safe Area``也适用在 tvOS 上。如果要将你的 app 和内容放到 tvOS 上，你可能会遇到各种各样的尺寸的屏幕，在某些情况如下图，我们顶部的标题太靠近顶部边缘，可能因此被遮挡掉一部分。

![顶部标题被截取](https://user-gold-cdn.xitu.io/2018/6/14/163fc40196f0c205?w=1181&h=664&f=jpeg&s=48826)

这个时候我们要调整我们的内容，让它处于``Safe Area``之中。``Safe Area``代表``storyboard``中的这块浅绿色的区域，你只要将你视图中的约束设置到``Safe Area``中，那它就安全了。

![浅绿色的安全区域](https://user-gold-cdn.xitu.io/2018/6/14/163fc405f53d1ecf?w=1181&h=664&f=jpeg&s=30403)

然后用一张的美丽背景图像填充剩余视图空间，妈妈在也不用担心我们的内容被导航条和标签栏挡住了。如下图，它们只会听话地呆在深色矩形方框内。

![添加背景](https://user-gold-cdn.xitu.io/2018/6/14/163fc4092561a258?w=1181&h=664&f=jpeg&s=61297)

### 4.3 开启 Safe Area Layout Guide

开启``Safe Area Layout Guide``也十分简单，打开我们的``storyboard``，进入 file inspector 标签页，然后找到``Use Safe Area Layout Guides``并且勾选上。你会发现每个控制器中都会出现一个``Safe Area``视图，然后你就可以像其他视图一样，将约束连向它。

![开启Safe Area](https://user-gold-cdn.xitu.io/2018/6/14/163fc40cb51106d3?w=680&h=554&f=jpeg&s=124199)

``Safe Area Layout Guide``是 UIView 的新特性，在之前的版本中顶部和底部的``Layout Guides``之间的矩形区域将与新的``Safe Area``相匹配，他们可以互相转换，如果在 iOS 11 的故事板中启用``Safe Area``，在你选中``afe Area Layout Guides``勾选框时 Xcode 将会自动升级你的约束。总而言之，在 Xcode 9 的故事板中使用``Safe Area Layout Guide``，将向下兼容 iOS 老版本的。

## 5. 比例定位（Proportional positioning）

接下来我们要谈一谈，关于如何将一个视图定位在其 superview 的布局技术，我们将其称之为 Proportional positioning ，即按 **比例定位** 。在安卓的布局技术中也有类似的功能，它的应用面广泛且实用，相信未来的开发中一定会频繁使用到。

### 5.1 比例布局

假设现在我有一个需求，要将我们 app 中的卡片高度定位在其 superview 高度的70%。也许你会有几种方式可以实现上述需求，但是现在我要用一个最直接的方式来实现它，便是我现在要介绍得的使用 spacerview 的方法。

![高度为superview高度的70%](https://user-gold-cdn.xitu.io/2018/6/14/163fc4135ac31946?w=1181&h=664&f=jpeg&s=22227)

从对象库拖出一个视图，只是一个普通的``UIView``。为它添加约束后设置隐藏，这样就不会渲染它，让它做一个安静的美男子，这样它就成为你需要定位的视图的参照物。而且这种技术也可以组合使用，灵活搭配，这里有另一个例子，我有一个场景，要遵守1/5，2/5和2/5的比例，然后他们以这些比例填充满整个屏幕。下面让我们来看看如何做到的。

![比例1:2:2](https://user-gold-cdn.xitu.io/2018/6/14/163fc4181e205e25?w=1181&h=664&f=jpeg&s=28934)

### 5.2 构建 SpacerView

如下图中我们已经有一个基本的布局。我有一个 label 和一个 image ,已经添加了基本约束。
当我选中它们，如果你仔细看，你会注意到它左边和右边的约束是蓝色的，但顶部和底部是红色的。这意味着我们还需要添加一些约束来定位。无论何时在 Interface Builder 画布中看到红色，那只可能是两种情况，要么你的约束太少，位置是不确定的，或者设置了太多的约束，其中一部分是冲突的。

![出现冲突](https://user-gold-cdn.xitu.io/2018/6/14/163fc41cf0f7bda9?w=454&h=804&f=png&s=12154)

我知道是因为我没有对垂直方向位置进行固定。所以接下来我们要通过创建 spacerview 来实现。拖一个``UIView``出来。首先我们将其隐藏，这样不会浪费性能进行绘制，我们为其添加好上方、左侧以及宽度的约束后，我们还没有设置其高度约束，我们要为其设置等高约束，使其高度与 superview 高度相，如下图效果。

![spacerview](https://user-gold-cdn.xitu.io/2018/6/14/163fc420f6df8541?w=454&h=794&f=png&s=11009)

接着让我们查看等高约束的属性，修改比例为70%，设置成功后你会发现``spacerview``的高度已经缩减了，下一步设置 Second Item 即比例参考对象视图为 ``Safe Area``，这样我们的``spacerview``就已经设置好了。

![设置为 Safe Area 70%高度](https://user-gold-cdn.xitu.io/2018/6/14/163fc42411257052?w=340&h=331&f=png&s=19131)

### 5.3 对齐到 Baseline

现在我们要将我们卡片视图底部与``spacerview``底部对齐，所以我们添加了底部对齐的约束，如果我想要是``spacerview``与我们的卡片文案的``baseline``对齐怎么办？选中约束后转到属性检查器，选择 FirstItem 选项，选中``First Baseline``即可。

![对齐baseline](https://user-gold-cdn.xitu.io/2018/6/14/163fc42722a53dbe?w=340&h=348&f=png&s=33277)

重新运行后得到了我想要的效果。

![占比 70%](https://user-gold-cdn.xitu.io/2018/6/14/163fc4299487c066?w=473&h=807&f=png&s=9305)

所以当你需要在 Interface Builder 中使用这个比例定位技术时，使用``spacerview``能够帮助你达到期望。**但是一定要将这些视图标记为隐藏**，使它们不会被渲染，但又能协助你进行布局，使你能够定位你的内容。如果你是用编程方式进行布局，可以使用``UILayout Guide``来完成，你可以将其用作等效于``spacerviews``。

## 6. Stack view 自适应布局（Stack view adaptive layout）

让我们一起来看看最后一种我们要布局的视图，如下图，你能看到 app 中展示了一个自适应布局的页面，上方是一个4x4的网格排列，底部有一个 label 。

![竖屏效果](https://user-gold-cdn.xitu.io/2018/6/14/163fc42e50484079?w=454&h=890&f=jpeg&s=29210)

当我旋转手机的时候，出现了一些不一样的东西。它仍然会显示一个4x4的网格，但它有一个文本视图出现在右边的位置。

![横屏效果](https://user-gold-cdn.xitu.io/2018/6/14/163fc43313820a34?w=907&h=462&f=jpeg&s=47826)

### 6.1 竖屏布局

这一切是怎么做到的呢？让我们来看看如何对 Interface Builder 中的 stackview 进行自适应布局。首先看下图中最外层是一个垂直的 stackview ，从上到下分成三行，第一行和第二行都是包含两张图片的水平 stackview ，第三行是一个 label 。就如你看到的，他们高度是相等的，我们可以通过 **Alignment**，**Distribution**，和 **Spacing** 等属性来进行调整，以达到你想要的布局。 stackview 有一个非常赞的地方，就是它能帮你管理被包含的视图的约束，这样你只要添加很少的约束。

![三行布局](https://user-gold-cdn.xitu.io/2018/6/14/163fc43942635030?w=454&h=790&f=png&s=57954)

接下来让我们选中所有的 stackview ，在``Distribution``选项中选择``fill equally`` 实现平均分布，在``Spacing``选项中我们可以手动输入我们想要的间距，另外系统也提供了标准间距选项给我们，点击输入框右边的倒三角就会出现一个``Use Standard Value``选项，直接选中即可。

![间距属性](https://user-gold-cdn.xitu.io/2018/6/14/163fc43ef4af4228?w=340&h=194&f=png&s=11695)

下一步我要确保这些图像是正方形的，我们直接选中第一张图片，为其添加一个宽高比为1:1的约束，添加后你会发现出现一些冲突，这是因为在满足填充满整个屏幕和三行平均分布的同时，无法保证图片比例达到1:1。

![宽高比例约束冲突](https://user-gold-cdn.xitu.io/2018/6/14/163fc44158e8aabf?w=454&h=809&f=png&s=56094)

所以我们要做一些改变，我们将 stackview 到底部的固定约束修改成大于等于，这样出现的冲突就解决了，也达到了我想要的效果。

![调整底部约束](https://user-gold-cdn.xitu.io/2018/6/14/163fc4441d495392?w=454&h=811&f=png&s=56421)

### 6.2 横屏布局

当我们将设备旋转到横屏状态，我们预期的效果是在右边有一个  textview ，而底部并没有 label  ，为了更接近我们预期效果我们需要把底部的 label 先隐藏。我们要如何才能做到在竖屏中显示，在横屏中隐藏呢？

![横屏](https://user-gold-cdn.xitu.io/2018/6/14/163fc4471984d89c?w=750&h=454&f=png&s=56248)

在全新的 Xcode9 中的隐藏属性，可以为不同``size class``分别设置显示或隐藏。转到 label 的``hidden``属性，你会发现勾选按钮左边有一个加号，它让这一切变得轻松简单。点击后在弹出的界面中``Width``选择``any``，在横屏的时候，``Height``选择``compact``，因为在横屏的时候它的高度是紧凑的。

![隐藏属性变量](https://user-gold-cdn.xitu.io/2018/6/14/163fc449aaa34895?w=680&h=219&f=png&s=25341)

做完这些点击``add  variation``，并且在``hidden``属性下面找到刚刚设置的隐藏属性并且勾选中它，你会发现 label 被隐藏了，如果你切换成竖屏，又会显示出来。

接下来继续添加一个 textview ，为了做到这点，我们要在最外层套一个水平排列的 stackview ，然后将 textview 加入 stackview 中，并且为新建的 stackview 添加约束，其中底部约束就如同之前设置为大于等于。这样就得到了我们横屏中需要的效果。

![加入textView](https://user-gold-cdn.xitu.io/2018/6/14/163fc44cbf15c4b1?w=719&h=454&f=png&s=65839)

当我们切换到竖屏时 textview 仍然显示了，我们要在竖屏时隐藏它，就像之前一样转到隐藏菜单，并添加一个变量，``Width``选择``any``，``Height``选择``Regular``，然后将其标记为隐藏，这样在竖屏时,  textview 就不再显示了，就此达到了我们期望的效果。

![竖屏时需要隐藏 textView](https://user-gold-cdn.xitu.io/2018/6/14/163fc44f890b3496?w=454&h=795&f=png&s=42622)

在使用 stackview 的时候，我们可以使用``Alignment``，``Distribution``，``Spacing``这些属性帮助我们布局。还有嵌套使用，只需要加入很少的约束，我们仅仅需要在你对宽高比例有要求的时候，通过宽高比例约束来获得我们想要的比例。令人惊喜的是，Xcode 9 中的隐藏属性是可分级的，它非常适合与 stackview 搭配使用，而且随着``size class``的变化的隐藏属性是向下兼容的。

![](https://user-gold-cdn.xitu.io/2018/6/14/163fc64a32ad9354?w=500&h=889&f=gif&s=4241781)

## 总结

到这里我们已经看完了``Autolayout``相关的的六种技术，在你构建 app 的时候有了更多的布局手段，这些技术能够使你的界面看起来非常美观，结构清晰，并且自适应布局，在日常开发中会经常使用到。我已经迫不及待地想看到更多人使用这些技术了。

## Demo

GitHub：[FindMyDates](https://github.com/xietao3/FindMyDates)

## 参考

- 视频地址：[WWDC 2017 Session 412 - Auto Layout Techniques  in Interface Builder](https://developer.apple.com/videos/play/wwdc2017/412/)
- PDF地址：[WWDC 2017 Session 412 - Auto Layout Techniques  in Interface Builder](https://devstreaming-cdn.apple.com/videos/wwdc/2017/412icy0vh6ays/412/412_auto_layout_techniques_in_interface_builder.pdf)




