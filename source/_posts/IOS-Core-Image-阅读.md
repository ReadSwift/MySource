---
title: IOS Core Image 阅读
date: 2016-03-10 22:41:29
tags: [Core Image]
categories: [编程]
---
> 核心图像是一个强大的框架，允许您轻松地将筛选器应用到图像，例如修改自然饱和度、 色调或暴露。它使用 GPU （或 CPU，用户可定义） 来处理图像数据，速度非常快。足够的快的做的视频帧实时处理 ！
>
> 核心图像筛选器可以堆叠在一起，一次将多个效果应用到图像或视频的帧。当多个筛选器被层叠在一起他们是有效的因为他们创建一个修改过的单个过滤器，被应用于图像，而不是通过每个筛选器，一次一个的处理图像。

#### OverView

开始之前你需要了解的Class：

1. **CIContext**。所有图像处理的核心是在CIContext上，这有点类似于Core Graphics或 OpenGL 的Context。

2. **CIImage**。这个类包含图片数据，可以从UIImage、图片文件或者像素数据创建。

3. **CIFilter**。filter class有一个定义了特定的属性字典筛选器。过滤器是自然饱和度滤镜，颜色反转过滤器、裁剪的过滤器，以及更多。

#### 介绍

使用时在项目中导入**Core Image framework**。

##### 基本图片滤波	

1. **Create a CIImage object**. CIImage 可以通过这些方法来初始化：

   `imageWithURL:`

   ` imageWithData: `

    `imageWithCVPixelBuffer: `

    `andimageWithBitmapData:bytesPerRow:size:format:colorSpace: `

   `imageWithURL:`

2. **Create a CIContext**.	CIContext 可以是 基于 CPU 或 GPU。CIContext 可以（也应该）被重用，所以你不应该创建它一遍又一遍，但当输出 CIImage 对象时，您将总是需要一个。

3. **Create a CIFilter**	. 当你创建筛选器时，它取决于您正在使用的筛选器上配置的一系列属性。

4. **Get the filter output**.  筛选器会给你返回一个CIImage的 图片。你可以把它转换成UIImage 。

```objc
NSString *filePath = [[NSBundle mainBundle] pathForResource:@"image"ofType:@"png"];
NSURL *fileNameAndPath = [NSURL fileURLWithPath:filePath];
//1
CIImage *beginImage = [CIImage imageWithContentsOfURL:fileNameAndPath];
// 2
CIContext *context = [CIContext contextWithOptions:nil];
//The CISepiaTone filter takes only two values, the kCIInputImageKey (a CIImage) and the @”inputIntensity”, a float value, wrapped in an NSNumber (using the new literal syntax), between 0 and 1. Here you give that value 0.8.
CIFilter *filter = [CIFilter filterWithName:@"CISepiaTone" keysAndValues: kCIInputImageKey, beginImage, @"inputIntensity", @0.8, nil];
CIImage *outputImage = [filter outputImage];
// 3
CGImageRef cgimg = [context createCGImage:outputImage fromRect:[outputImage 
                                                                extent]];
UIImage *newImage = [UIImage imageWithCIImage:cgimg];
CGImageRelease(cgimg);
```

##### 更改筛选器(**filter**)的值														

 你可以更改filter的**inputIntensity**的值来改变图片的滤镜。																																																				

```objc
- (void)imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary *)info {
  [self dismissViewControllerAnimated:YES completion:nil]; 
  UIImage *gotImage = [info objectForKey:UIImagePickerControllerOriginalImage]; 
  UIImageOrientation orientation = gotImage.imageOrientation;
  beginImage = [CIImage imageWithCGImage:gotImage.CGImage]; 
  [filter setValue:beginImage forKey:kCIInputImageKey];
  [self amountSliderValueChanged:self.amountSlider];
}
```

```objc
- (IBAction)savePhoto:(id)sender { // 1
	CIImage *saveToSave = [filter outputImage]; // 2
	CIContext *softwareContext = [CIContext
	contextWithOptions: @{kCIContextUseSoftwareRenderer : @(YES)} ];
	// 3
	CGImageRef cgImg = [softwareContext createCGImage:saveToSave fromRect:		
                    	[saveToSave extent]];
  //根据方向活的图片
    UIImage *newImage = [UIImage imageWithCGImage:cgimg scale:1.0 orientation:orientation];
	// 4
	ALAssetsLibrary* library = [[ALAssetsLibrary alloc] init]; 
    [library  writeImageToSavedPhotosAlbum:cgImg metadata:[saveToSave properties] 
     completionBlock: ^(NSURL *assetURL, NSError *error) {
	// 5
	CGImageRelease(cgImg);
	}]; 
}
```

##### 其他的filters

CIFilter 有100多个API，你可以用这个方法查看都有什么样的filter存在

`[CIFilter filterNamesInCategory:kCICategoryBuiltIn] `

每个filter都有一个字典属性的字典信息。

```objc
-(void)logAllFilters {
	NSArray *properties = [CIFilter filterNamesInCategory:kCICategoryBuiltIn]; 
  	NSLog(@"%@", properties);
	for (NSString *filterName in properties) {
	CIFilter *fltr = [CIFilter filterWithName:filterName];
	NSLog(@"%@", [fltr attributes]);
   }
}
```

##### 更复杂的filter Chains

```objc
- (CIImage *)oldPhoto:(CIImage *)img withAmount:(float)intensity { 

  //1 建立棕褐色的筛选器
  CIFilter *sepia = [CIFilter filterWithName:@"CISepiaTone"]; 
  [sepia setValue:img forKey:kCIInputImageKey];
  [sepia setValue:@(intensity) forKey:@"inputIntensity"];

  //2 随机筛选器创建一个随机噪声模式，它不使用任何参数。你会使用这种噪声模式将纹理添加到您最后的老  照片看。
  CIFilter *random = [CIFilter filterWithName:@"CIRandomGenerator"];

  //3 你正在改变随机噪声发生器的输出，你想要使它的灰色点亮,设置输入image key 这是一种简便方法链一个过滤器的输出到下一个输入
  CIFilter *lighten = [CIFilter filterWithName:@"CIColorControls"];
  [lighten setValue:random.outputImage forKey:kCIInputImageKey];
  [lighten setValue:@(1 - intensity) forKey:@"inputBrightness"];
  [lighten setValue:@0.0 forKey:@"inputSaturation"];

  //4 使用 imageByCroppingToRect 方法将输出 CIImage 到剪切到指定rect
  CIImage *croppedImage = [lighten.outputImage imageByCroppingToRect:[beginImage 
                                                                      extent]];

  //5 联合输出模式 类似photoshop的选项
  CIFilter *composite = [CIFilter filterWithName:@"CIHardLightBlendMode"];
  [composite setValue:sepia.outputImage forKey:kCIInputImageKey];
  [composite setValue:croppedImage forKey:kCIInputBackgroundImageKey];

  //6
  CIFilter *vignette = [CIFilter filterWithName:@"CIVignette"];
  [vignette setValue:composite.outputImage forKey:kCIInputImageKey];
  [vignette setValue:@(intensity * 2) forKey:@"inputIntensity"];
  [vignette setValue:@(intensity * 30) forKey:@"inputRadius"];

  //7
  return vignette.outputImage; 
}
```

#### Compositing filters(混合)

复合**filter**能创建几乎任何可能的预期的效果,介绍两个筛选器	**CISourceAtopComposition**和**CISepiaTone**

```objc
- (IBAction)maskModeOn:(id)sender { 
  //1
  CIImage *maskImage = [CIImage imageWithCGImage: [UIImage                                                                                             		imageNamed:@"sampleMaskPng.png"].CGImage];

  //2 创建白色 虚变的filter
  CIFilter *maskFilter = [CIFilter filterWithName:@"CISourceAtopCompositing" 
                          keysAndValues:kCIInputImageKey, beginImage, 
                          @"inputBackgroundImage", maskImage, nil];

  //3
  CGImageRef cgImg = [context createCGImage:maskFilter.outputImage fromRect:
                      [maskFilter.outputImage extent]];

  //4
  [self.imageView setImage:[UIImage imageWithCGImage:cgImg]]; 

  //5
  CGImageRelease(cgImg);
}

-(CIImage *)maskImage:(CIImage *)inputImage {
  UIImage *mask = [UIImage imageNamed:@"sampleMaskPng.png"];
  CIImage *maskImage = [CIImage imageWithCGImage:mask.CGImage]; 
  CIFilter *maskFilter = [CIFilter
                          filterWithName:@"CISourceAtopCompositing" 
                          keysAndValues:kCIInputImageKey, inputImage, 
                          kCIInputBackgroundImageKey, maskImage, nil];

  return maskFilter.outputImage;
}
```

#### Combining a filtered image with another

```objc
-(CIImage *)addBackgroundLayer:(CIImage *)inputImage {
  // This method is going to take an input image you supply and composite it with the bryce.png file. The compositing method will simply place the bryce image underneath the supplied image. If the supplied image is not transparent, you won’t see bryce at all.
  UIImage *backImage = [UIImage imageNamed:@"bryce.png"]; 
  CIImage *bg = [CIImage imageWithCGImage:backImage.CGImage]; 
  CIFilter *sourceOver = [CIFilter
                          filterWithName:@"CISourceOverCompositing" 
                          keysAndValues:kCIInputImageKey, inputImage, 
                          @"inputBackgroundImage", bg, nil];

  return sourceOver.outputImage; 
}
//Resizing your input image
```

#### Drawing a mask	

```objc
- (CGImageRef)drawMyCircleMask:(CGPoint)location { 
  NSUInteger width = (NSUInteger)self.imageView.frame.size.width; 
  NSUInteger height =(NSUInteger)self.imageView.frame.size.height; 
  CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
  NSUInteger bytesPerPixel = 4;
  NSUInteger bytesPerRow = bytesPerPixel * width;
  NSUInteger bitsPerComponent = 8;

  CGContextRef cgcontext = CGBitmapContextCreate(NULL, width, height, 
                                                 bitsPerComponent, bytesPerRow, 
                                                 colorSpace,
                                                 kCGImageAlphaPremultipliedLast | 
                                                 kCGBitmapByteOrder32Big); 
  CGColorSpaceRelease(colorSpace); 
  CGContextSetRGBFillColor(cgcontext, 1, 1, 0.5, .7); 
  CGContextFillEllipseInRect(cgcontext,
                             CGRectMake(location.x - 25, location.y - 25, 50.0, 
                                        50.0)); 
  CGImageRef cgImg = CGBitmapContextCreateImage(cgcontext); 
  CGContextRelease(cgcontext);

  return cgImg;
}

//我们可以简化上面的方法
-(void)setupCGContext {
  NSUInteger width = (NSUInteger)
  self.imageView.frame.size.width; 
  NSUInteger height = (NSUInteger)self.imageView.frame.size.height;
  CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB(); 
  NSUInteger bytesPerPixel = 4;

  NSUInteger bytesPerRow = bytesPerPixel * width;
  NSUInteger bitsPerComponent = 8;
  cgcontext = CGBitmapContextCreate(NULL, width, height,
                                    bitsPerComponent, bytesPerRow, colorSpace,
                                    kCGImageAlphaPremultipliedLast | 
                                    kCGBitmapByteOrder32Big); 
  CGColorSpaceRelease(colorSpace);
}

- (CGImageRef)drawMyCircleMask:(CGPoint)location reset:(BOOL)reset {

  if (reset) {
    CGContextClearRect(cgcontext, CGRectMake(0, 0, 320,
                                             self.imageView.frame.size.height));
  }

  CGContextSetRGBFillColor(cgcontext, 1, 1, 1, .7); 
  CGContextFillEllipseInRect(cgcontext, CGRectMake(location.x - 25, location.y - 
                                                   25, 50.0, 50.0)); 
  CGImageRef cgImg = CGBitmapContextCreateImage(cgcontext); 
  return cgImg;
}
```

#### Face detection

该系统可以检测到图像中是否有一个或多个面孔，其中包括脸周围的边界框的位置。

```objc
- (void)hasFace:(CIImage *)image { 
  CIDetector *faceDetector = [CIDetector detectorOfType:CIDetectorTypeFace 
                              context:nil options: @{ CIDetectorAccuracy: 
                                                     CIDetectorAccuracyHigh} ];
  NSArray *features = [faceDetector featuresInImage:image]; 
  NSLog(@"%@", features);
  for (CIFaceFeature *f in features) {
    NSLog(@"%f, %f", f.leftEyePosition.x, f.leftEyePosition.y);
  } 
}
//Note: Orientation is an issue with face detection. If you provide an image that is in an orientation other than UIImageOrientationUp you will need to rotate it to that orientation before handing it to the face detector.

// You’re going to be using these boxes to mark the eyes and mouth of your faces.
- (CIImage *)createBox:(CGPoint)loc color:(CIColor *)color { 
  //The first filter creates a solid color image, the CIConstantColorGenerator takes only one input in the form of a CIColor. This filter creates a solid color image with an infinite extent, which means it goes on forever unless you tell it how to crop it.
  CIFilter *constantColor = [CIFilter
                             filterWithName:@"CIConstantColorGenerator" 
                             keysAndValues:@"inputColor", color, nil];

  CIFilter *crop = [CIFilter filterWithName:@"CICrop"];
  [crop setValue:constantColor.outputImage forKey:kCIInputImageKey];
  [crop setValue:[CIVector vectorWithCGRect:CGRectMake(loc.x - 3, loc.y - 3, 6, 6)] 
   forKey:@"inputRectangle"];

  return crop.outputImage; 
}

//You’re basically using Core Image filters to draw boxes.
- (CIImage *)filteredFace:(NSArray *)features onImage: (CIImage *)inputImage {
  CIImage *outputImage = inputImage; 
  for (CIFaceFeature *f in features) {
    if (f.hasLeftEyePosition) {
      CIImage *box = [self createBox:
                      CGPointMake(f.leftEyePosition.x, f.leftEyePosition.y) color:
                      [CIColor colorWithRed:1.0 green:0.0 blue:0.0 alpha:0.7]];
      outputImage = [CIFilter filterWithName: @"CISourceAtopCompositing" 
                     keysAndValues:kCIInputImageKey, box, 
                     kCIInputBackgroundImageKey, outputImage, nil].outputImage;
    }

    if (f.hasRightEyePosition) {
      CIImage *box = [self createBox:CGPointMake(f.rightEyePosition.x, 
                                                 f.rightEyePosition.y) color:
                      [CIColor colorWithRed:1.0 green:0.0 blue:0.0 alpha:0.7]];
      outputImage = [CIFilter filterWithName: @"CISourceAtopCompositing" 
                     keysAndValues:kCIInputImageKey, box, 
                     kCIInputBackgroundImageKey, outputImage, nil].outputImage;
    }

    if (f.hasMouthPosition) {
      CIImage *box = [self createBox:CGPointMake(f.mouthPosition.x, 
                                                 f.mouthPosition.y) color:[CIColor                                                                                                                                                 
                   colorWithRed:0.0 green:0.0 blue:1.0 alpha:0.7]];

      outputImage = [CIFilter filterWithName: @"CISourceAtopCompositing" 
                     keysAndValues:kCIInputImageKey, box, 
                     kCIInputBackgroundImageKey, outputImage, nil].outputImage;

    } 
  }
return outputImage; 
}
```

#### Looking at other filters																																																																													

```objc
-(void)loadFiltersArray {
  //CIAffineTransform helps you apply a CGAffineTransform to an image. These
transforms can scale, rotate, and skew an image.
  CIFilter *affineTransform = [CIFilter filterWithName:
                             @"CIAffineTransform" keysAndValues:kCIInputImageKey, 
                             beginImage, @"inputTransform", 
                             [NSValue valueWithCGAffineTransform: 
                              CGAffineTransformMake(1.0, 0.4, 0.5, 1.0, 0.0, 0.0)], 
                             nil];
  
//CIStraightenFilter is used to fix an image take with the camera tilted slightly.
You give it an inputAngle property to determine the amount of rotation to correct.
  CIFilter *straightenFilter = [CIFilter filterWithName: @"CIStraightenFilter" 
                                keysAndValues:kCIInputImageKey, beginImage, 
                                @"inputAngle", @2.0, nil];
// CIVibrance will increase the color intensity
  CIFilter *vibrance = [CIFilter filterWithName:@"CIVibrance" 
                        keysAndValues:kCIInputImageKey, beginImage, @"inputAmount", 
                        @- 0.85, nil];

  //CIColorControls allow you to change brightness, contrast, and saturation.
  CIFilter *colorControls = [CIFilter filterWithName:@"CIColorControls" 
                             keysAndValues:kCIInputImageKey,
                             beginImage, @"inputBrightness", @-0.5, 
                             @"inputContrast", @3.0, @"inputSaturation", @1.5, 
                             nil];
//CIColorInvert will invert the colors. It only takes one parameter, inputImage.
  CIFilter *colorInvert = [CIFilter filterWithName:@"CIColorInvert" 
                           keysAndValues:kCIInputImageKey, beginImage, nil];
//CIHighlightShadowAdjust will allow you to change the values of the highlights and shadows in your image.
  CIFilter *highlightsAndShadows = [CIFilter filterWithName: 
                                    @"CIHighlightShadowAdjust" 
                                    keysAndValues:kCIInputImageKey, beginImage, 
                                    @"inputShadowAmount", @1.5, 
                                    @"inputHighlightAmount", @0.2, nil];
//CIGuassianBlur blurs the input image. A blur is a common operation in image processing, but is resource intensive.
  CIFilter *blurFilter = [CIFilter filterWithName:@"CIGaussianBlur" 
                          keysAndValues:kCIInputImageKey, beginImage, nil];

  //CIPixellate samples the image at square intervals and creates larger pixels from that color.
  CIFilter *pixellate = [CIFilter filterWithName:@"CIPixellate" 
                         keysAndValues:kCIInputImageKey, beginImage, @"inputScale", 
                         @10.0, nil];
// CIBumpDistortion warps the image at a location in a circular way.
  CIFilter *bumpDistortion = [CIFilter filterWithName:@"CIBumpDistortion" 
                              keysAndValues:kCIInputImageKey, beginImage, 
                              @"inputScale", @0.8, @"inputCenter", [CIVector                                                                                                                                                                                                    
							vectorWithCGPoint:CGPointMake(160, 120)],
                              @"inputRadius", @150, nil];
//CIMinimumComponent Returns a grayscale image from min(r,g,b).
  CIFilter *minimum = [CIFilter filterWithName:@"CIMinimumComponent" 
                       keysAndValues:kCIInputImageKey, beginImage, nil];

  CIFilter *circularScreen = [CIFilter filterWithName:@"CICircularScreen" 
                              keysAndValues:kCIInputImageKey, beginImage, 
                              @"inputWidth", @10.0, nil];
//CICircularScreen create a circular pattern of lines where the darker the underlying image, the thicker the lines.
  filtersArray = @[affineTransform, straightenFilter, vibrance, colorControls, 
                   colorInvert, highlightsAndShadows, blurFilter, pixellate, 
                   bumpDistortion, minimum, circularScreen];

  filterIndx = 0; 
}


//例子
- (IBAction)rotateFilter:(id)sender {
  CIFilter *filt = [filtersArray objectAtIndex:filterIndx]; CGImageRef imgRef = 
    [context createCGImage:filt.outputImage
     fromRect:[filt.outputImage extent]];

  [self.imageView setImage:[UIImage imageWithCGImage:imgRef]]; 
  CGImageRelease(imgRef);

  filterIndx = (filterIndx + 1) % [filtersArray count]; 
  [self.filterTitle setText:[[filt attributes] 
                             valueForKey:@"CIAttributeFilterDisplayName"]];
}
```

#### Auto-Enhancement	

苹果有另一个灵巧的特性叫auto-enhancement. 你调用CIImage会翻译一个filters 数组包括vibrance, highlights and shadows, red eye reduction, flesh tone, etc																																												

```objc
- (IBAction)autoEnhance:(id)sender {
  CIImage *outputImage = beginImage;
  NSArray *ar = [beginImage autoAdjustmentFilters]; 
  
  for (CIFilter *fil in ar) {
    [fil setValue:outputImage forKey:kCIInputImageKey]; 
    outputImage = fil.outputImage;
    NSLog(@"%@", [[fil attributes] valueForKey:@"CIAttributeFilterDisplayName"]);
  }

  [filter setValue:outputImage forKey:kCIInputImageKey]; 
  CGImageRef cgimg = [context createCGImage:outputImage
                      fromRect:[outputImage extent]];

  UIImage *newImage = [UIImage imageWithCGImage:cgimg scale:1.0 
                       orientation:orientation];
  self.imageView.image = newImage; CGImageRelease(cgimg);
}
```