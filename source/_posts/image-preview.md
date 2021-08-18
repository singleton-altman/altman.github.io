---
title: iOS图片预览
tags:
  - iOS
  - image preview
  - Photos
date: 2021-08-18 11:32:25
---


**背景**
    原先项目内的图片预览框架用的是一个魔改的某开源框架, 里面的代码又杂又乱, 各种乱拷贝代码. 这周公司的项目任务已经做完, 就决定给这个框架自己重写一下.
<!-- more -->   
**效果**
    - 支持单图/多图点开按顺序预览
    - 支持单张放大/缩小
    - 支持从头部和尾部插入列表
    - 支持滑动回收预览
    - 图片可以是`GIF`
    - 支持将图片保存入系统相册

    <video 
    id="video" 
    controls="" 
    preload="none" 
    poster="http://om2bks7xs.bkt.clouddn.com/2017-08-26-Markdown-Advance-Video.jpg">
      <source 
      id="mp4" 
      src="http://blog.loktar.com.cn/image_preview.mp4" type="video/mp4">
    </video>
**实现**
    - 主体使用`UIScrollView`, 然后利用`UIScrollViewDelegate`的 `- scrollViewDidZoom:` 和 `- viewForZoomingInScrollView:`配合, 实现滚动、滑动和放大缩小功能.
        - 此处需要注意的是: 层级结构为: `main UIScrollView` -> [`sub UIScrollView` -> `UIView` -> `SDAnimationImageView`], 不然在拖拽时候会出现显示异常
        - 在 `sub UIScrollView` 的 `- scrollViewDidZoom:`中需要重设 `- viewForZoomingInScrollView: ` 的位置.
    ```
    CGFloat ratio = MIN(self.bounds.size.width / size.width, self.bounds.size.height / size.height);
    CGFloat W = ratio * size.width;
    CGFloat H = ratio * size.height;
    CGRect imageFrame = CGRectMake((self.bounds.size.width - W) / 2, (self.bounds.size.height - H) / 2, W, H);
    self.containerView.frame = imageFrame;
    self.imageView.frame = CGRectMake(0, 0, imageFrame.size.width, imageFrame.size.height);
    ```
    - 给主体控件上添加`UITapGestureRecognizer`手势, 实现在拖拽的时候进行回收/重新布局. 主要的
    ```
    - (void)handlePanGestureRecognizer:(UIPanGestureRecognizer *)sender {
        static LRSImagePreviewZoomingImageView *currentView = nil;
        if (sender.state == UIGestureRecognizerStateBegan) {
            /// 在手势开始的时候获取当前正在拖拽的自视图
            currentView = ...;
        }
        if (currentView) {
            if (sender.state == UIGestureRecognizerStateEnded) {
                /// 重置
                currentView = nil;
            } else {
                /// 拖拽动画
                CGPoint p = [sender translationInView:self];
                CGAffineTransform transform = CGAffineTransformMakeTranslation(p.x, p.y);
                CGFloat value = 1 - (MAX(fabs(p.x), fabs(p.y))) / 1000;
                transform = CGAffineTransformScale(transform, value, value);
                currentView.transform = transform;
            }
        }
    }
    ```
    - 因为还需要从图片点击位置出现/回收视图, 所以还需要记录弹出预览图的点击位置.
    - 往头部插入数据的时候, 记得设置`Scroview`的偏移量
    - 需要支持`GIF`, 所以我们引用了`SDWebImage`
    - 图片保存系统相册, 我们使用了系统的`Photos`框架
    ```
    + (void)creationRequestWithData:(NSData *)data options:(nullable PHAssetResourceCreationOptions *)options completionHandler:(void (^)(BOOL, NSError * _Nullable))handler {
        PHPhotoLibrary *library = [PHPhotoLibrary sharedPhotoLibrary];
        [library performChanges:^{
            [[PHAssetCreationRequest creationRequestForAsset] addResourceWithType:PHAssetResourceTypePhoto data:data options:options];
        } completionHandler:handler];
    }
    ```
**注意点**
    - 坐标换算
    `CGRect rect = [A convertRect:B.frame fromView:B.supverView]`
    得到的`rect`就是`B`相对于`A`的新坐标.
**demo地址**
    已经上传`github`, [enjoy](https://github.com/aioser/LRSImagePreview)

