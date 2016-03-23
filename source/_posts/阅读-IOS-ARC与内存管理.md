---
title: 阅读 IOS ARC与内存管理
date: 2016-03-03 20:11:22
tags: [ARC]
categories: 编程
---
### 对象生命周期    

```objc
id obj = [array objectAtIndex: 0];
[array removeObjectAtIndex: 0];
NSLog(@"%@", obj);
```
#### 手动管理内存

array拥有对象obj 引用计数器为1，移除对象后引用计数器为0，再次输出会crash。在移除之前可以retain该对象:    

`[obj retain];`  

##### ARC

在ARC中obj 变量为强引用计数器自动加1，即使数组移除后这个对象依然被obj引用着。

Properties  总结：
1. **strong**.    
   This is a synonym for the old “retain”. A strong property becomes an owner of the object it points to.  
2. **weak**.  
   This is a property that represents a weak pointer. It will automatically be set to nil when the pointed-to object is destroyed. Remember, use this for outlets.  
3. **unsafe_unretained**.  
   This is a synonym for the old “assign”. You use it only in exceptional situations and when you want to target iOS 4. More about this later.  
4. **copy**.  
   This is still the same as before. It makes a copy of the object and creates a strong relationship.  

5. **assign**.  
   You’re no longer supposed to use this for objects, but you still use it forprimitive values such as BOOL, int, and float.

#### ARC用法示例    

1. 在**delloc**你仍然可以使用实例变量，因为还没被释放。直到**dellac** **return**。

2. 基本数据类型在属性中仍然使用**assign**.  

3. **Core Foundation** to **Objective-C**  

   **__bridge** keyword to indicate that **ARC** remains in charge.
   ```objc
   //在最新地LLVM编译器中 __bridge 可能没有必要，因为编译器足够聪明可以推断出需要的地方加 __bridge
   - (NSString *)escape:(NSString *)text {
     return (NSString *)CFBridgingRelease( CFURLCreateStringByAddingPercentEscapes(NULL,(__bridge CFStringRef)text,NULL,(CFStringRef)@"!*'();:@&=+$,/?%#[]", CFStringConvertNSStringEncodingToEncoding(NSUTF8StringEncoding)));
   }
   ```
   如果你使用  **__bridge** 代替 **CFBridgeRelease** 则会导致内存泄露
   **__bridge_transfer**: Give ARC ownership  (**CFBridgingRelease()**)  
   **__bridge_retained**: Relieve ARC of its ownership(** CFBridgingRetain()**)    

   如果你使用 **CFBridgingRetain()**  从 object to **Core Foundation** 那么 ARC 将不在负责释放它：
   ```objc
   NSString *s1 = [[NSString alloc] initWithFormat:@"Hello, %@!", name];
   CFStringRef s2 = CFBridgingRetain(s1); // do something with s2// . . .CFRelease(s2);
   ```
   Remember, anywhere you call a Core Foundation function named **Create**, **Copy**, or **Retain****** you must do **CFBridgingRelease**() to safely transfer the value to ARC.*    

    *Not all Objective-C and Core Foundation objects that sound alike are toll-free bridged. For example, CGImage and UIImage cannot be cast to each another, and neither can CGColor and UIColor. The following page lists the types that can be used interchangeably: <http://bit.ly/j65Ceo>*

4. 从 Objective-C object to **void ***  
   你需要使用 **__bridge** 例如:
   ```objc
   MyClass *myObject = [[MyClass alloc] init];
   [UIView beginAnimations:nil context:(__bridge void *)myObject];
   ```
   在动画代理方法你需要转换:
   ```objc
   - (void)animationDidStart:(NSString *)animationID context:(void *)context
   {
   MyClass *myObject = (__bridge MyClass *)context; ...
   }
   ```
5. Delegates and **Weak** Properties

   在设计对象属性时候可以参考下面:
   - **parent** pointing to a **child**: strong

   - **child** pointing to a **parent**: weak

     ```objc
     @property (nonatomic, strong) NSString *artistName;
     @property (nonatomic, weak) IBOutlet UINavigationBar *navigationBar;
     ```

     **outlet** 在 **nib** 中 并不是 top-levelobjects 所以设置为**weak** 会在低内存自动释放。

     ```objc
     //Note: 在IOS5 和 IOS6 有细微的差距. 在 iOS 5 有可能针对以前卸载的视图控制器如果变得再次可见时会再次调用您的 viewDidLoad 方法. 如果你想要在iOS 6 也能如此实现, 可以按如下方法:
     - (void)didReceiveMemoryWarning {
       [super didReceiveMemoryWarning]; _soundEffect = nil;
       if ([self isViewLoaded] && self.view.window == nil) {
       NSLog(@"forcing my view to unload");
       self.view = nil; 
       }
     }
     ```

6. **Unsafe_unretained**
   除了strong 和 weak 还有 unsafe_unretained，你通常不会想使用它。因为它会只想一个已经不存在的对象，它存在是为了兼容IOS 4. (因为IOS 4没有weak 指针)
7. **Blocks**

   在ARC以前Blocks 在stack上面被初始化，如果你想要在另一个作用域引用当前的block。你必须使用copy 或者Block_copy() 它到堆(heap)上，现在ARC自动给你做了这些。

   例如自定义一个**AnimatedView**：

   ```objc
   #import <UIKit/UIKit.h>
   typedef void (^AnimatedViewBlock)(CGContextRef context, CGRect
     rect, CFTimeInterval totalTime, CFTimeInterval deltaTime);
   @interface AnimatedView : UIView
   @property (nonatomic, copy) AnimatedViewBlock block;
   @end
   ```
   ```objc
   #import <QuartzCore/QuartzCore.h> 
   #import "AnimatedView.h"
   @implementation AnimatedView {
     NSTimer *_timer; 
     CFTimeInterval _startTime; 
     CFTimeInterval _lastTime;
   }
   - (id)initWithCoder:(NSCoder *)aDecoder {
     if ((self = [super initWithCoder:aDecoder])) {
     _timer = [NSTimer scheduledTimerWithTimeInterval:0.1 target:self
     selector:@selector(handleTimer:) userInfo:nil repeats:YES];
     _startTime = _lastTime = CACurrentMediaTime();
    }
   return self; 
   }
   - (void)dealloc {
     NSLog(@"dealloc AnimatedView");
     [_timer invalidate]; 
   }

   - (void)handleTimer:(NSTimer*)timer {
     [self setNeedsDisplay]; 
   }

   - (void)drawRect:(CGRect)rect {
     CFTimeInterval now = CACurrentMediaTime();
     CFTimeInterval totalTime = now - _startTime; 
     CFTimeInterval deltaTime = now - _lastTime; 
     _lastTime = now;
     if (self.block != nil) self.block(UIGraphicsGetCurrentContext(), rect,
     totalTime, deltaTime);
   }
   ```

   这个类有一个block属性, NSTimer 强引用他的target，正好是AnimatedView自己本身，这就造成循环引用。除非你明确释放其中某一个，否则他们一直引用对方。怎么打破尼？
   - 可以实现一个方法让使用这个类的对象来调用，如下：
     ```objc
     - (void)stopAnimation {
     [_timer invalidate], 
       _timer = nil;
     }
     ```

   - 你可能还有其他的方法，但是你不能简单的指定 NSTimer 为 __weak，因为这会造成AnimatedView不在拥有timer对象，但这对你并没有好处，这个定时器仍然被 the run loop 拥有，所以会导致AnimatedView一直存在不被释放。直到你使定时器无效。

   ```objc
   //在viewDidLoad中调用block
   - (void)viewDidLoad {
     [super viewDidLoad]; 
     self.navigationBar.topItem.title = self.artistName;
     UIFont *font = [UIFont boldSystemFontOfSize:24.0f]; 
     CGSize textSize = [self.artistName sizeWithFont:font];

     self.animatedView.block = ^(CGContextRef context, CGRect rect, CFTimeInterval totalTime, CFTimeInterval deltaTime) {
     NSLog(@"totalTime %f, deltaTime %f", totalTime, deltaTime);
     CGPoint textPoint = CGPointMake( (rect.size.width - textSize.width)/2,    
                                     (rect.size.height - textSize.height)/2);

     [self.artistName drawAtPoint:textPoint withFont:font];
     }
   }
   ```
   如果你这样写的话AnimatedView不会释放，而且ViewController也不会释放。注意在block内部引用了**self.artistName**，所以block 引用了VC，有几种解决方法其中一种就是不引用 any properties, instance variables, or methods from the block。
   ```objc
   //使用局部变量text不会导致循环引用
   NSString *text = self.artistName;
   self.animatedView.block = ^(CGContextRef context, CGRect rect, CFTimeInterval totalTime, CFTimeInterval deltaTime)
   {
     CGPoint textPoint = CGPointMake((rect.size.width - textSize.width)/2, (rect.size.height - textSize.height)/2);
     [text drawAtPoint:textPoint withFont:font]; 
   };

   //如果你不得不在Block中使用self可以这样用 weakSelf 不会导致retain
   __weak DetailViewController *weakSelf = self;

   self.animatedView.block = ^(CGContextRef context, CGRect rect, CFTimeInterval totalTime, CFTimeInterval deltaTime) {
     DetailViewController *strongSelf = weakSelf;//你仍然需要检查这个weak指针是否alive，所以最好指定weak pointer 给一个新的变量， strongSelf 你临时把它变成一个强引用直到你的对象不在使用它。
     if (strongSelf != nil) {
   NSLog(@"totalTime %f, deltaTime %f", totalTime, deltaTime);
   CGPoint textPoint = CGPointMake( (rect.size.width - textSize.width)/2,
   (rect.size.height - textSize.height)/2);
   [strongSelf.artistName drawAtPoint:textPoint withFont:font];
   } 
   };
   ```

   在大多数情况下你在block内部可以不必转化为StrongSelf，然而在这种情况下，这么使用会Crash，
   ```objc
   __weak DetailViewController *weakSelf = self;
   self.animatedView.block = ^(CGContextRef context, CGRect rect, CFTimeInterval totalTime, CFTimeInterval deltaTime)
   {
   ...
         //nil -> artistName 会crash
   [weakSelf->_artistName drawAtPoint:textPoint withFont:font];
   }
   ```

8. **Singletons**

   ```objc
   #import "GradientFactory.h" 
   @implementation GradientFactory
   + (id)sharedInstance {
     static GradientFactory *sharedInstance; 
     if (sharedInstance == nil) {
     sharedInstance = [[GradientFactory alloc] init]; 
     }

     return sharedInstance; 
   }

   //因为返回一个 creat 的 core foundation 对象 所以你得负责释放它，当你使用的时候、
   - (CGGradientRef)newGradientWithColor1:(UIColor *)color1 color2:(UIColor *)color2 color3:(UIColor *)color3 midpoint:(CGFloat)midpoint {
     //UIColor and CGColorRef are NOT toll-free bridged, NSArray can only hold Objective-C objects, not Core Foundation objects, you have to cast that CGColorRef back to an id， That works because all Core Foundation object types are toll-free bridged with NSObject。but only in regards to memory handling. So you can treat a CGColor as an NSObject, but not as a UIColor.
     NSArray *colors = @[(id)color1.CGColor, (id)color2.CGColor,(id)color3.CGColor        ];
     const CGFloat locations[3] = { 0.0f, midpoint, 1.0f };
     CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB(); 

     //Of course, the CGGradientCreateWithColors() function doesn’t accept an NSArray so you need to cast colors to a CFArrayRef to do toll-free bridging the other way around, In this case you still want to keep ARC responsible for releasing the NSArray object, so a simple __bridge cast serves to indicate that no transfer of ownership is taking place。
     CGGradientRef gradient = CGGradientCreateWithColors(
     colorSpace, (__bridge CFArrayRef)colors, locations);       
     CGColorSpaceRelease(colorSpace);

     return gradient; 
   }
   ```
   在下面这种情况下会Crash：
   ```objc
   CGColorRef cgColor1 = [[UIColor alloc] initWithRed:1 green:0 blue:0 alpha:1].CGColor;
   CGColorRef cgColor2 = [[UIColor alloc] initWithRed:0 green:1 blue:0 alpha:1].CGColor;
   CGColorRef cgColor3 = [[UIColor alloc] initWithRed:0 green:0 blue:1 alpha:1].CGColor;
   NSArray *colors = @[ (__bridge id)cgColor1, (__bridge id)cgColor2,
   (__bridge id)cgColor3 ];
   ```
   The reason is simple, you create a UIColor object that is not autoreleased but retained (because you call alloc + init). As soonas there are no strong pointers to this object, it gets deallocated. Because a CGColorRef is not an Objective-C object, the cgColor1 variable does not qualify as astrong pointer. The new UIColor object is immediately released after it is createdand cgColor1 points at garbage memory. Yikes!

   Change **viewDidLoad** to:
   ```objc
   - (void)viewDidLoad {
     [super viewDidLoad]; self.navigationBar.topItem.title = self.artistName;
     UIFont *font = [UIFont boldSystemFontOfSize:24.0f]; 
     CGSize textSize = [self.artistName sizeWithFont:font];

     float components[9];
     NSUInteger length = [self.artistName length];
     NSString* lowercase = [self.artistName lowercaseString];

     for (int t = 0; t < 9; ++t) {
       unichar c = [lowercase characterAtIndex:t % length];
       components[t] = ((c * (10 - t)) & 0xFF) / 255.0f; }

     UIColor *color1 = [UIColor colorWithRed:components[0] green:components[3] 
                        blue:components[6] alpha:1.0f];
     UIColor *color2 = [UIColor colorWithRed:components[1] green:components[4] 
                        blue:components[7] alpha:1.0f];
     UIColor *color3 = [UIColor colorWithRed:components[2] green:components[5] 
                        blue:components[8] alpha:1.0f];

     __weak DetailViewController *weakSelf = self;

     self.animatedView.block = ^(CGContextRef context, CGRect rect, CFTimeInterval 
                                 totalTime, CFTimeInterval deltaTime)
     {
       DetailViewController *strongSelf = weakSelf; if (strongSelf != nil) {
         CGPoint startPoint = CGPointMake(0.0, 0.0);
         CGPoint endPoint = CGPointMake(0.0, rect.size.height);  
         CGFloat midpoint = 0.5f + (sinf(totalTime))/2.0f;

         CGGradientRef gradient = [[GradientFactory sharedInstance]
                                   newGradientWithColor1:color1 color2:color2 
                                   color3:color3 midpoint:midpoint];

         CGContextDrawLinearGradient(context, gradient, startPoint, endPoint, 
                                     kCGGradientDrawsBeforeStartLocation | 
                                     kCGGradientDrawsAfterEndLocation);

         CGGradientRelease(gradient);

         CGPoint textPoint = CGPointMake( (rect.size.width - textSize.width)/2, 
                                         (rect.size.height - textSize.height)/2);

         [strongSelf.artistName drawAtPoint:textPoint withFont:font];
   } };
   }
   ```

   If your singleton can be used from multiple threads then the simple sharedInstance accessor method does not suffice. A more sturdy implementation is this:

   ```objc
   + (id)sharedInstance {
     static GradientFactory *sharedInstance; 
     static dispatch_once_t done;

     dispatch_once(&done, ^ {
     sharedInstance = [[GradientFactory alloc] init]; 
     });

     return sharedInstance;
   }
   ```

9. **Autorelease**

   在**ARC**中**autorelease** 和 **autorelease pool** 仍然在使用作为一种语言结构(@autoreleasepool)而不是一个**class**。

   使用包含 init，new，copy 或 mutableCopy 方法的名称的时候，返回一个retained对象。在没有变量引用的情况下会自动释放，如果是一个autorelease对象，那么只会autorelease pool 释放的时候去deallocated。在ARC以前你会手动调用release NSAutoreleasePool 对象，现在在**@autoreleaseplool** block结尾处自动释放。
   ```objc
   @autoreleasepool {
     NSString *s = [NSString stringwithFormat:...];
   }
   // the string object is deallocated when the code gets here
   ```

   ```objc
   NSString *s;
   @autoreleasepool {
     s = [NSString stringWithFormat:. . .];
   }
   // the string object is still alive here.  As long as s is in scope, the string object stays alive.
   ```

   有一个方式能使上面的例子对象自动释放：
   ```objc
   //The special __autoreleasing keyword tells the compiler that this variable’s contents may be autoreleased. Now it no longer is a strong pointer and the string object will be deallocated at the end of the @autoreleasepool block
   __autoreleasing NSString *s;
   @autoreleasepool {
     s = [NSString stringWithFormat:. . .];
   }
   // the string object is deallocated at this point 
   NSLog(@"%@", s); // crash!
   ```

   你可能不会用到 __autoreleaseing，但你可能遇到out-parameters 或者 pass-by-reference，特别是这种带有(NSError**)的方法参数。

   ```objc
   #import <QuartzCore/QuartzCore.h>

   - (NSString *)documentsDirectory {
     NSArray *paths = NSSearchPathForDirectoriesInDomains( NSDocumentDirectory, NSUserDomainMask, YES);
     return [paths lastObject]; 
   }

   - (IBAction)coolAction {
     UIGraphicsBeginImageContext(self.animatedView.bounds.size);
     [self.animatedView.layer renderInContext:UIGraphicsGetCurrentContext()];
     UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
     UIGraphicsEndImageContext();

     NSData *data = UIImagePNGRepresentation(image);
     if (data != nil){
     NSString *filename = [[self documentsDirectory] 
                             stringByAppendingPathComponent:@"Cool.png"];
       NSError *error;
       //If there is some kind of error, the method will create a new NSError object and store that into your error variable. This is also known as an out-parameter and it’s often used to return more than one value from a method or function. you’ll see that this NSError ** parameter is actually specified as NSError *__autoreleasing *
       if (![data writeToFile:filename options:NSDataWritingAtomic error:&error]){
     NSLog(@"Error: %@", error);
   }
     }
     [self.delegate detailViewController:self didPickButtonWithIndex:0];
   }
   ```

   `NSError *error;`

   then you implicitly say that the error variable is **__strong**. However, if you the npass the address of that variable into the writeToFile method, you want to treat itas an __autoreleasing variable. These two statements cannot both be true; a variable is either strong or autoreleasing, not both at the same time.

   To resolve this situation the compiler makes a new temporary variable. Under the hood the above code actually looks like this:

   ```objc
   NSError *error;
   __autoreleasing NSError *temp = error;
   BOOL result = ![data writeToFile:filename options:NSDataWritingAtomic 
   error:&temp];
   error = temp;
   if (!result) {
     NSLog(@"Error: %@", error);
   }

   //Generally speaking this extra temporary variable is no big deal. But if you want to avoid it you can write the code as follows:
   __autoreleasing NSError *error;
   if (![data writeToFile:filename options:NSDataWritingAtomic error:&error]) {
   NSLog(@"Error: %@", error);
   }
   ```

   写你自己的方法需要返回一个 out-parameter, 你可以这样写:

   ```objc
   - (NSString *)fetchKeyAndValue:(__autoreleasing NSNumber **)value {
   NSString *theKey; 
     NSString *theValue;
   // do whatever you need to do here
   *value = theValue;
   return theKey; 
   }
   ```

   你可以这样调用：
   ```objc
   NSNumber *value;
   NSString *key = [self fetchKeyAndValue:&value];
   ```

   当然你不需要特别指定__autoreleaseing 在 out-parameter，如果你不想要把对象放进autorelease pool，你可以这样声明out-parameter:
   ```objc
   - (NSString *)fetchKeyAndValue:(__strong NSNumber **)value {
   ...
   }
   ```

   需要注意的一些事情是一些API使用自己的autorelease pool，例如 NSDictionary's enumerateKeysAndObjectsUsingBlock:方法在block调用前会设置autorelease pool，那意味着你在block内部创建的自动释放对象会被 NSDictionary's  pool释放。通常你会想要这么做除了下面的情况：
   ```objc
   - (void)loopThroughDictionary:(NSDictionary *)d error:(NSError **)error {
     [d enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop){
       // do stuff . . .
       if (there is some error && error != nil) {
         *error = [NSError errorWithDomain:@"MyError" code:1 userInfo:nil];
       } }];
   }
   ```

   **error**变量作为**out-parameter**将会自动释放，因为enumerateKeysAndObjectsUsingBlock:  有它自己的autorelease pool，任何新建的error对象，在方法返回之前将会被释放。你可以使用一个临时的strong变量去引用这个NSError 对象：

   ```objc
   - (void)loopThroughDictionary:(NSDictionary *)d error:(NSError **)error{
   __block NSError *temp;
   [d enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop){
   // do stuff . . .
   if (there is some error) {
   temp = [NSError errorWithDomain:@"MyError" code:1 userInfo:nil];
   if (error != nil) *error = temp;
       }
    }];
    }
   ```

   如果你要创建大量对象你可以创建自己的autoreleasepool：
   ```objc
   for (int i = 0; i < 1000000; i++) {
   @autoreleasepool {
   NSString *s = [NSString stringWithFormat:. . .];
   // do stuff . . .
   }
   }
   //Note: If you’re creating a new thread you also still need to wrap your code in an autorelease pool using the @autorelease syntax. The principle hasn’t changed, just the syntax.
   ```

   ARC做了大量优化，通常你调用stringWithFormat方法返回一个autorelease object， 但是你用一个strong 本地变量引用它，当出了本地作用域，这个string object会被释放。所以不需要把string object放进autorelease pool。

   *Note: There is no such thing as autoreleasing a Core Foundation object. The principle of autorelease is purely an Objective-C thing. Some people havefound ways to autorelease CF objects anyway by first casting them to an id,then calling autorelease, and then casting them back again:*

   `return (CGImageRef)[(id)myImage autorelease];`

   *That obviously won’t work anymore because you cannot call autorelease withARC. If you really feel like you need to be able to autorelease Core Foundationobjects, then you can make your own CFAutorelease() function and put it in afile that gets compiled with -fno-objc-arc.*
10. **didReceiveMemoryWarning**

    现在你可以这么写去释放不在window中显示的视图：

    ```objc
    - (void)didReceiveMemoryWarning {
      [super didReceiveMemoryWarning];
      if ([self isViewLoaded] && self.view.window == nil) {
        self.view = nil;
        sortedNames = nil;
        sortedValues = nil;
      }
    }
    ```

    ​