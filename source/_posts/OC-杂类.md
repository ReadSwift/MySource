---
title: OC 杂类
date: 2016-03-15 19:47:54
tags: Object-C Tip
categories:
---
> 介绍一些OC使用中的小诀窍。混排，不分顺序。

##### 自定义下标

你可以实现这个方法来自定义你的类对象下标方法。

```objc
- (id)objectAtIndexedSubscript:(NSUInteger)idx;
- (void)setObject:(id)obj atIndexedSubscript:(NSUInteger)idx;
- (void)setObject:(id)obj forKeyedSubscript:(id <NSCopying>)key;
```

更多语法信息请查看：

[ http://clang.llvm.org/docs/ObjectiveCLiterals.html](http://clang.llvm.org/docs/ObjectiveCLiterals.html)

 http://clang.llvm.org/docs/LanguageExtensions.html

##### Debug试图约束关系

调用私有**API _autolayoutTrace**：

添加约束前记得**translatesAutoresizingMaskIntoConstraints**设置为NO

```objc
@interface UIWindow (AutoLayoutDebug) 
  + (UIWindow *)keyWindow;
  - (NSString *)_autolayoutTrace;
@end

//And adding these two methods somewhere inside the @implementation body:
 - (void)viewDidAppear:(BOOL)animated {
  [super viewDidAppear:animated];
  NSLog(@"%@", [[UIWindow keyWindow] _autolayoutTrace]); 
}

- (void)didRotateFromInterfaceOrientation 
  (UIInterfaceOrientation)fromInterfaceOrientation {
  [super didRotateFromInterfaceOrientation:fromInterfaceOrientation];
  NSLog(@"%@", [[UIWindow keyWindow] _autolayoutTrace]);
}
```

你会发现有歧义的约束。当你移除一些约束，又添加一些约束的时候，你需要调用**setNeedsUpdateConstraints**来触发它。

```objc
- (CGSize)systemLayoutSizeFittingSize:(CGSize)targetSize //返回最合适的尺寸，ios6
- (CGSize)intrinsicContentSize //返回view的自然尺寸，ios6
- (void)invalidateIntrinsicContentSize //使内容尺寸无效化，ios6
```

如果你需要改变约束做动画的话可以这么做：

```objc
[UIView animateWithDuration:0.3f animations:^ {
  [self.view layoutIfNeeded]; 
}];
```

如果你删除旧的增加新的约束，需要在block中调用layoutIfNeeded。

##### 自定义布局Layout

自定布局layout时候会调用**intrinsicContentSize**来获取它的size，你可以在自定义的view里面实现该方法。那么自定义的view也支持auto layout.

如果你只想计算content的宽度。可以如下返回：

```objc
- (CGSize)intrinsicContentSize {
  return CGSizeMake(mySize.width, UIViewNoIntrinsicMetric); 
}
```

这样的话自动布局只会计算高度，宽度由我自己定义。

你可以限定你自已定义View的content区域，增加下面这个方法：

```objc
- (UIEdgeInsets)alignmentRectInsets {
  return UIEdgeInsetsMake(10, 10, 10, 10); 
}
```

你可以通过这个方法来观察我自己view的内容区域：

```objc
- (CGSize)systemLayoutSizeFittingSize:(CGSize)targetSize //返回最合适的尺寸，ios6
- (CGSize)intrinsicContentSize //返回view的自然尺寸，ios6
- (void)invalidateIntrinsicContentSize //使内容尺寸无效化，ios6
```

```objc
- (void)drawRect:(CGRect)rect {
  [[UIColor redColor] setFill];
  CGRect alignmentRect = [self alignmentRectForFrame:self.bounds];
  UIRectFill(alignmentRect);
  NSLog(@"Alignment rect: %@", NSStringFromCGRect(alignmentRect));
}

//你可以增加这个来改变尺寸
mySize.width += 30.0f; 
mySize.height += 30.0f;
[self invalidateIntrinsicContentSize]; //使内容尺寸无效
[self setNeedsDisplay];// add this
```

如果你自定义的试图包含子视图，你可以使用autolayout设置subviews，最好在initwithframe：或者initwithcoder：设置。你需要重写`+requiresConstraintBasedLayout`返回YES。（*返回view是否是约束布局模式*）

```objc
+ (BOOL)requiresConstraintBasedLayout //返回view是否是约束布局模式,ios6
- (BOOL)translatesAutoresizingMaskIntoConstraints //返回一个BOOL，判断自动布局是否可为转换约束布局，ios6
- (void)setTranslatesAutoresizingMaskIntoConstraints:(BOOL)flag //设置在约束布局系统中是否把自动布局转换为约束布局，ios6
```

当然在约束不改变的情况下工作的很好，如果你改变了约束可以调用下面的方法去更新约束：

`updateConstraints`

`layoutSubviews`

你可以在**UIViewController**中实现`updateViewConstraints`去动态的更新按钮上的约束。

通过调用`setNeedsUpdateConstraints`或者`updateConstraintsIfNeeded`告诉Auto Layout 一个view视图的约束需要刷新，它会首先在View中调用**updateConstraints**，接着调用相关viewcoontroller的**updateViewConstraints**. 所以你可以在updateConstraints中集中管理你的constraints. 你可以动态移除子试图在layoutSubviews。

#####  测量约束布局(**Measuring in Constraint-Based Layout**)

理解**Content Hugging** 和 **Content Compression Resistance**。

这两个属性对有intrinsic content size的控件（例如button，label）非常重要。通俗的讲，具有intrinsic content size的控件自己知道（可以计算）自己的大小，例如一个label，当你设置text，font之后，其大小是可以计算到的。关于intrinsic content size官方的解释:

*Custom views typically have content that they display of which the layout system is unaware. Overriding this method allows a custom view to communicate to the layout system what size it would like to be based on its content. This intrinsic size must be independent of the content frame, because there’s no way to dynamically communicate a changed width to the layout system based on a changed height, for example.*

```objc
//UIView中关于Content Hugging 和 Content Compression Resistance的方法有：
- (UILayoutPriority)contentCompressionResistancePriorityForAxis
  (UILayoutConstraintAxis)axis //待补充，ios6
- (void)setContentCompressionResistancePriority:(UILayoutPriority)priority forAxis:(UILayoutConstraintAxis)axis //待补充，ios6
- (UILayoutPriority)contentHuggingPriorityForAxis:(UILayoutConstraintAxis)axis //待补充，ios6
- (void)setContentHuggingPriority:(UILayoutPriority)priority forAxis:(UILayoutConstraintAxis)axis //待补充，ios6
```

大概的意思就是设置优先级的。Hugging priority 确定view有多大的优先级阻止自己变大。Compression Resistance priority确定有多大的优先级阻止自己变小，很抽象。

**content Hugging**就是要维持当前view在它的optimal size（intrinsic content size），可以想象成给view添加了一个额外的width constraint，此constraint试图保持view的size不让其变大：

view.width <= optimal size.

此constraint的优先级就是通过上面的方法得到和设置的，content Hugging默认为250.

**Content Compression Resistance**就是要维持当前view在他的optimal size（intrinsic 
content size），可以想象成给view添加了一个额外的width 
constraint，此constraint试图保持view的size不让其变小：

view.width >= optimal size.

此默认优先级为750.

示例：

```objc
button1 = [UIButton buttonWithType:UIButtonTypeRoundedRect];
button1.translatesAutoresizingMaskIntoConstraints = NO;
[button1 setTitle:@"button 1 button 2" forState:UIControlStateNormal];
[button1 sizeToFit];

[self.view addSubview:button1];

//距离view左边100
NSLayoutConstraint *constraint = [NSLayoutConstraint constraintWithItem:button1 attribute:NSLayoutAttributeLeading relatedBy:NSLayoutRelationEqual toItem:self.view attribute:NSLayoutAttributeLeading multiplier:1.0f constant:100.0f];
[self.view addConstraint:constraint];

//距离view上边100
constraint = [NSLayoutConstraint constraintWithItem:button1 attribute:NSLayoutAttributeTop relatedBy:NSLayoutRelationEqual toItem:self.view attribute:NSLayoutAttributeTop multiplier:1.0f constant:100.0f];
[self.view addConstraint:constraint];

//距离view右边150
constraint = [NSLayoutConstraint constraintWithItem:button1 attribute:NSLayoutAttributeTrailing relatedBy:NSLayoutRelationEqual toItem:self.view attribute:NSLayoutAttributeTrailing multiplier:1.0f constant:-150.0f];
constraint.priority = 751.0f;//设置优先级
[self.view addConstraint:constraint];

//注意右边的constraint的优先级我们设置的为751.0，此时autolayout系统会去首先满足此constraint，再去满足Content Compression Resistance（其优先级为750，小于751）。
```



