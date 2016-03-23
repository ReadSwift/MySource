---
title: UICollectionView 介绍
date: 2016-03-21 20:28:36
tags: IOS-UICollectionView
categories:
---
>  在IOS iOS 6中，苹果引进新的类叫UICollectionView ，可以使你很简单的创建漂亮自定义布局和试图切换.

#### Anatomy of a UICollectionViewController

##### UICollectionViewController组成

UICollectionViewController包含了几个重要的关键组件。

- **Supplementary View**

  如果你有附加的信息不想要展示在cell里面但是在collection view里面，你可以使用supplementaty views。 通常被用作 sections 的headers 和 footers。

- **UICollectionViewCell**

  和UITableVIewCell相似，这些cell组成collection view的内容。可以通过IB或者代码创建。

- **UICollectionView**

  主要视图被展示的地方类似与UITableView的功能。

- **Decoration View**

  如果你想要在UICollectionView的外表添加一些附加的信息（不包含有用的数据），你可以使用这个装饰。

- **UICollectionViewLayout**

  UICollectionView不知道任何关于建立cell到屏幕上，而这个任务由UICollectionViewLayout处理，它通过在UICollectionView设置代理方法把cell放在合适的位置，（ Layouts can be swapped out during runtime and theUICollectionView can even automatically animate switching from one layout toanother!）

- **UICollectionViewFlowLayout**

  你可以实现UICollectionViewLayout的子类去实现你自定义的layouts，苹果提供了一个基本的『flow-based』的layout叫UICollectionViewFlowLayout。像一个网格视图排列。

##### 实用例子

```objc
@interface ViewController () <UITextFieldDelegate>
@property(nonatomic, weak) IBOutlet UIToolbar *toolbar; 
@property(nonatomic, weak) IBOutlet UIBarButtonItem *shareButton; @property(nonatomic, weak) IBOutlet UITextField *textField;
- (IBAction)shareButtonTapped:(id)sender;
@end
  
  
//ViewDidLoad
  self.view.backgroundColor = [UIColor colorWithPatternImage:[UIImage                                                                                                                
imageNamed:@"bg_cork.png"]];
UIImage *navBarImage = [[UIImage imageNamed:@"navbar.png"] 
                        resizableImageWithCapInsets:UIEdgeInsetsMake(27, 27, 27,
                                                                     27)];
[self.toolbar setBackgroundImage:navBarImage 
 forToolbarPosition:UIToolbarPositionAny barMetrics:UIBarMetricsDefault];
UIImage *shareButtonImage = [[UIImage imageNamed:@"button.png"] 
                             resizableImageWithCapInsets:UIEdgeInsetsMake(8, 8, 8, 
                                                                          8)];
[self.shareButton setBackgroundImage:shareButtonImage forState:UIControlStateNormal 
 barMetrics:UIBarMetricsDefault];
UIImage *textFieldImage = [[UIImage imageNamed:@"search_field.png"] 
                           resizableImageWithCapInsets:UIEdgeInsetsMake(10, 10, 10, 
                                                                        10)];
[self.textField setBackground:textFieldImage];

[self.collectionView registerClass:[UICollectionViewCell class]
 forCellWithReuseIdentifier:@"MY_CELL"];

//Preparing for the UICollectionView

#pragma mark - UICollectionView Datasource
// 1
- (NSInteger)collectionView:(UICollectionView *)view numberOfItemsInSection:
(NSInteger)section {
  NSString *searchTerm = self.searches[section];
  return [self.searchResults[searchTerm] count];
}

// 2
- (NSInteger)numberOfSectionsInCollectionView: (UICollectionView *)collectionView {
  return [self.searches count]; 
}

// 3
- (UICollectionViewCell *)collectionView:(UICollectionView *)cv 
  cellForItemAtIndexPath:(NSIndexPath *)indexPath {
    UICollectionViewCell *cell = [cv 
                              dequeueReusableCellWithReuseIdentifier:@"FlickrCell " 
                                  forIndexPath:indexPath];
    cell.backgroundColor = [UIColor whiteColor];
    return cell; 
  }
// 4
- (UICollectionReusableView *)collectionView: (UICollectionView *)collectionView viewForSupplementaryElementOfKind:(NSString *)kind atIndexPath:(NSIndexPath *)indexPath {
  FlickrPhotoHeaderView *headerView = [collectionView 
                                       dequeueReusableSupplementaryViewOfKind:
                                       UICollectionElementKindSectionHeader 
                                       withReuseIdentifier:@"FlickrPhotoHeaderView" 
                                       forIndexPath:indexPath];

  NSString *searchTerm = self.searches[indexPath.section]; 
  [headerView setSearchText:searchTerm];

  return headerView;
}

#pragma mark - UICollectionViewDelegate
- (void)collectionView:(UICollectionView *)collectionView didSelectItemAtIndexPath:(NSIndexPath *)indexPath {
  // TODO: Select Item
}
- (void)collectionView:(UICollectionView *)collectionView 
  didDeselectItemAtIndexPath:(NSIndexPath *)indexPath {
    // TODO: Deselect item
}


#pragma mark – UICollectionViewDelegateFlowLayout
//There are more delegate methods you can implement than this

// 1 is responsible for telling the layout the size of a given cell. To do this, you must first determine which datasource you are looking at, since each photo could have different dimensions.
- (CGSize)collectionView:(UICollectionView *)collectionView layout:
(UICollectionViewLayout*)collectionViewLayout sizeForItemAtIndexPath:(NSIndexPath 
                                                                      *)indexPath {
  NSString *searchTerm = self.searches[indexPath.section]; 
  FlickrPhoto *photo = self.searchResults[searchTerm][indexPath.row];

  // 2 返回cell的size
  CGSize retval = photo.thumbnail.size.width > 0 ? photo.thumbnail.size :
  CGSizeMake(100, 100);
  
  retval.height += 35; 
  retval.width += 35; 
  return retval;
}

// 3 returns the spacing between the cells, headers, and footers.
- (UIEdgeInsets)collectionView:(UICollectionView *)collectionView layout:
(UICollectionViewLayout*)collectionViewLayout insetForSectionAtIndex:
(NSInteger)section {
  return UIEdgeInsetsMake(50, 20, 50, 20); 
}
```

#### 自定义Layout

##### A deeper dive into collection view

让我们先理解UICollectionView怎么维护Layouts。

##### Understanding the class hierarchy

![class hierarchy](http://7xs4cu.com1.z0.glb.clouddn.com/ios34EB94F9-3113-4E0E-8C07-7441C486889A.png)

你会看到所有的中心是UICollectionView，当你创建它的时候你会给它data source、delegate、layout。

##### Layout attributes

 UICollectionViewLayout 是一个基类，你需要override它的方法。

*Note: An abstract class is a class that you must subclass and can’t instantiatedirectly. In Objective-C, it’s not strictly true that you can’t instantiate aninstance of an abstract class, since there is no mechanism to enforce this.However, if the documentation specifies it as an abstract class, then youshould subclass it.*

当collection viw 想要知道怎么排列cells，supplementary views，以及 decoration views。它会询问layout attributes。

将这个数据封装的类称为 UICollectionViewLayoutAttributes，它描述了以下布局详细信息 ︰

1. **frame**: The location of the view within the coordinate system of the collectionview.
2. **center**: The center of the view within the coordinate system. This can be setalong with the size parameter instead of the frame parameter.
3.  **size**: The size of the view. This can be set along with the center parameterinstead of the frame parameter.
4.  **transform3D**:  A CATransform3D object which is applied to the view if you want torotate it, scale it, or transform it
5. **alpha**: The opacity of the view.
6. **zIndex**: The ordering of the view with respect to other views. This can be used todefine specifically whether the view should be on top of or below other views. Aview with a higher z-index will be displayed on top of a view with a lower z-index.
7. **hidden**:  Whether or not the view is hidden.

##### The lifecycle of a layout

