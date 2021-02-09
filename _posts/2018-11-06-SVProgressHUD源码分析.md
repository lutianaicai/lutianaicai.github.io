---
title: SVProgressHUD 源码分析
date: 2018-11-06 
categories: iOS
tags: 源码分析
---

众所周知 [SVProgressHUD](https://github.com/SVProgressHUD/SVProgressHUD) 是一个简洁易用的 HUD 库，我想探寻简洁易用背后的原理。

## Singleton

`SVProgressHUD` 同 `MBProgressHUD` 一样，都是 `UIView` 的子类，不同与 `MB` , `SV` 提供的是单例，这也是它简洁的一大因素。

```objc
+ (SVProgressHUD*)sharedView {
    static dispatch_once_t once;
    
    static SVProgressHUD *sharedView;
#if !defined(SV_APP_EXTENSIONS)
    dispatch_once(&once, ^{ sharedView = [[self alloc] initWithFrame:[[[UIApplication sharedApplication] delegate] window].bounds]; });
#else
    dispatch_once(&once, ^{ sharedView = [[self alloc] initWithFrame:[[UIScreen mainScreen] bounds]]; });
#endif
    return sharedView;
}
```

常规的单例创建，根据是否是 `App Extension` 进行了判断以确定 `Frame` 大小

## Show Methods

`SV` 不像 `MB` 可以有较高的自由度定制，所以为了简洁，暴露的全是类方法

```objc
+ (void)show;
+ (void)showWithStatus:(nullable NSString*)status;
+ (void)showProgress:(float)progress;
+ (void)showProgress:(float)progress status:(nullable NSString*)status;
```

没有列举全部，因为都不是重点，他们不添加图片的最终会调用主方法:

```objc
- (void)showProgress:(float)progress status:(NSString*)status { }
```

有图的调用:

```objc
- (void)showImage:(UIImage*)image status:(NSString*)status duration:(NSTimeInterval)duration { }
```

以无图为例:

```objc
- (void)showProgress:(float)progress status:(NSString*)status {
    __weak SVProgressHUD *weakSelf = self;
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        __strong SVProgressHUD *strongSelf = weakSelf;
        if(strongSelf){
            if(strongSelf.fadeOutTimer) {
                strongSelf.activityCount = 0;
            }
            
            // Set properties
            strongSelf.fadeOutTimer = nil;
            strongSelf.graceTimer = nil;
            ...
            
            // Choose the "right" indicator depending on the progress
            if(progress >= 0) {
                ...
                // Add ring to HUD
                if(!strongSelf.ringView.superview){
                    [strongSelf.hudView.contentView addSubview:strongSelf.ringView];
                }
                if(!strongSelf.backgroundRingView.superview){
                    [strongSelf.hudView.contentView addSubview:strongSelf.backgroundRingView];
                }
                ...
            } else {
                ...
                // Add indefiniteAnimatedView to HUD
                [strongSelf.hudView.contentView addSubview:strongSelf.indefiniteAnimatedView];
                if([strongSelf.indefiniteAnimatedView respondsToSelector:@selector(startAnimating)]) {
                    [(id)strongSelf.indefiniteAnimatedView startAnimating];
                }
                ...
            }
            
            // Fade in delayed if a grace time is set
            if (self.graceTimeInterval > 0.0 && self.backgroundView.alpha == 0.0f) {
                strongSelf.graceTimer = [NSTimer timerWithTimeInterval:self.graceTimeInterval target:strongSelf selector:@selector(fadeIn:) userInfo:nil repeats:NO];
                [[NSRunLoop mainRunLoop] addTimer:strongSelf.graceTimer forMode:NSRunLoopCommonModes];
            } else {
                [strongSelf fadeIn:nil];
            }
            
            ...
        }
    }];
}
```

首先是 `weakSelf` `strongSelf` 标准写法，老生常谈了避免循环引用以及提前释放。

然后设置一些属性，根据 progress 判断选择合适的指示符。

然后根据是否设置了 `gracetime` 调用或者延时调用 `fadeIn` 方法。

```objc
- (void)fadeIn:(id)data {
    // Update the HUDs frame to the new content and position HUD
    [self updateHUDFrame];
    [self positionHUD:nil];
    
    ...
    
    // Show if not already visible
    if(self.backgroundView.alpha != 1.0f) {
        ...
        
        __block void (^animationsBlock)(void) = ^{
            // Zoom HUD a little to make a nice appear / pop up animation
            self.hudView.transform = CGAffineTransformIdentity;
            
            // Fade in all effects (colors, blur, etc.)
            [self fadeInEffects];
        };
        
        __block void (^completionBlock)(void) = ^{
            // Check if we really achieved to show the HUD (<=> alpha)
            // and the change of these values has not been cancelled in between e.g. due to a dismissal
            if(self.backgroundView.alpha == 1.0f){
                // Register observer <=> we now have to handle orientation changes etc.
                [self registerNotifications];
                
                ...
                
                // Dismiss automatically if a duration was passed as userInfo. We start a timer
                // which then will call dismiss after the predefined duration
                if(duration){
                    self.fadeOutTimer = [NSTimer timerWithTimeInterval:[(NSNumber *)duration doubleValue] target:self selector:@selector(dismiss) userInfo:nil repeats:NO];
                    [[NSRunLoop mainRunLoop] addTimer:self.fadeOutTimer forMode:NSRunLoopCommonModes];
                }
            }
        };
        
        // Animate appearance
        if (self.fadeInAnimationDuration > 0) {
            // Animate appearance
            [UIView animateWithDuration:self.fadeInAnimationDuration
                                  delay:0
                                options:(UIViewAnimationOptions) (UIViewAnimationOptionAllowUserInteraction | UIViewAnimationCurveEaseIn | UIViewAnimationOptionBeginFromCurrentState)
                             animations:^{
                                 animationsBlock();
                             } completion:^(BOOL finished) {
                                 completionBlock();
                             }];
        } else {
            animationsBlock();
            completionBlock();
        }
        
        // Inform iOS to redraw the view hierarchy
        [self setNeedsDisplay];
    } else {
        // Update accessibility
        UIAccessibilityPostNotification(UIAccessibilityScreenChangedNotification, nil);
        UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, self.statusLabel.text);
        
        // Dismiss automatically if a duration was passed as userInfo. We start a timer
        // which then will call dismiss after the predefined duration
        if(duration){
            self.fadeOutTimer = [NSTimer timerWithTimeInterval:[(NSNumber *)duration doubleValue] target:self selector:@selector(dismiss) userInfo:nil repeats:NO];
            [[NSRunLoop mainRunLoop] addTimer:self.fadeOutTimer forMode:NSRunLoopCommonModes];
        }
    }
}
```

其实就是如果已经显示完全了就 `dismiss`，如果没有显示完全就先显示完全再 `dismiss`

## Dismiss Methods

同样 dismiss 暴露的也全部是类方法

* `+ (void)popActivity;`
* `+ (void)dismiss;`
* `+ (void)dismissWithCompletion:(nullable SVProgressHUDDismissCompletion)completion;`
* `+ (void)dismissWithDelay:(NSTimeInterval)delay;`
* `+ (void)dismissWithDelay:(NSTimeInterval)delay completion:(nullable SVProgressHUDDismissCompletion)completion;`

同样也是最终调用主方法：
`- (void)dismissWithDelay:(NSTimeInterval)delay completion:(SVProgressHUDDismissCompletion)completion`
这个方法其实逻辑跟 `show` 也差不多就是最终把 `controlView` `backgroundView` `budView` 都 remove 掉然后再把自己 remove，还有取消动画 `progress` 归零等

```objc
- (void)dismissWithDelay:(NSTimeInterval)delay completion:(SVProgressHUDDismissCompletion)completion {
    __weak SVProgressHUD *weakSelf = self;
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        __strong SVProgressHUD *strongSelf = weakSelf;
        if(strongSelf){
            ...
            
            __block void (^completionBlock)(void) = ^{
                // Check if we really achieved to dismiss the HUD (<=> alpha values are applied)
                // and the change of these values has not been cancelled in between e.g. due to a new show
                if(self.backgroundView.alpha == 0.0f){
                    // Clean up view hierarchy (overlays)
                    [strongSelf.controlView removeFromSuperview];
                    [strongSelf.backgroundView removeFromSuperview];
                    [strongSelf.hudView removeFromSuperview];
                    [strongSelf removeFromSuperview];
                    
                    // Reset progress and cancel any running animation
                    strongSelf.progress = SVProgressHUDUndefinedProgress;
                    [strongSelf cancelRingLayerAnimation];
                    [strongSelf cancelIndefiniteAnimatedViewAnimation];
                    
                    ...                    
                    // Run an (optional) completionHandler
                    if (completion) {
                        completion();
                    }
                }
            };
            
            // UIViewAnimationOptionBeginFromCurrentState AND a delay doesn't always work as expected
            // When UIViewAnimationOptionBeginFromCurrentState is set, animateWithDuration: evaluates the current
            // values to check if an animation is necessary. The evaluation happens at function call time and not
            // after the delay => the animation is sometimes skipped. Therefore we delay using dispatch_after.
            
            dispatch_time_t dipatchTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC));
            dispatch_after(dipatchTime, dispatch_get_main_queue(), ^{
                if (strongSelf.fadeOutAnimationDuration > 0) {
                    // Animate appearance
                    [UIView animateWithDuration:strongSelf.fadeOutAnimationDuration
                                          delay:0
                                        options:(UIViewAnimationOptions) (UIViewAnimationOptionAllowUserInteraction | UIViewAnimationCurveEaseOut | UIViewAnimationOptionBeginFromCurrentState)
                                     animations:^{
                                         animationsBlock();
                                     } completion:^(BOOL finished) {
                                         completionBlock();
                                     }];
                } else {
                    animationsBlock();
                    completionBlock();
                }
            });
            
            // Inform iOS to redraw the view hierarchy
            [strongSelf setNeedsDisplay];
        }
    }];
}
```

这里有一个小细节，注释也写的很清楚了。就是`UIViewAnimationOptionBeginFromCurrentState` 搭配 `delay` 总是有点跳动画的毛病，所以SV这里用 `dispatch_after` 来延时。

## 总结

总体来讲，[SVProgressHUD](https://github.com/SVProgressHUD/SVProgressHUD) 还是简洁易用为主，Github上给出的使用建议都是这种：

```objc
[SVProgressHUD show];
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    // time-consuming task
    dispatch_async(dispatch_get_main_queue(), ^{
        [SVProgressHUD dismiss];
    });
});
```

源码实现确实也不算复杂，个人还是比较喜欢用。


