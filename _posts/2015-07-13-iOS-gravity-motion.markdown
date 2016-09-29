---
layout:     post
title:      "iOS开发小Tips，在关闭自动旋屏的状态下检测屏幕方向"
subtitle:   "利用CoreMotion.frameWork来检测屏幕的方向。"
date:       2015-07-13
author:     "waynezxcv"
header-img: "img/post.png"
tags:
- something not important
---




## 利用UIDeviceOrientationDidChangeNotification来检测屏幕方向

一般情况下，我们可以通过注册通知UIDeviceOrientationDidChangeNotification来检测屏幕的方向变化。

```

[[NSNotificationCenter defaultCenter] addObserver:self
selector:@selector(orientationChanged:)
name:UIDeviceOrientationDidChangeNotification
object:nil];

- (void)orientationChanged:(NSNotification *)notification {
UIDeviceOrientation  orient = [UIDevice currentDevice].orientation;
switch (orient) {
case UIDeviceOrientationUnknown:{
//            NSLog(@"unknow");
}
break;
case UIDeviceOrientationPortrait:{
//            NSLog(@"portrait");
}
break;
case UIDeviceOrientationLandscapeLeft:{
//            NSLog(@"landscapeleft");
}
break;
case UIDeviceOrientationPortraitUpsideDown:{
//            NSLog(@"poraitupsidedown");
}
break;
case UIDeviceOrientationLandscapeRight:{
//            NSLog(@"landscaperight");
}
break;
case UIDeviceOrientationFaceUp:{
//            NSLog(@"faceUp");
}
break;
case UIDeviceOrientationFaceDown:{
//            NSLog(@"faceDown");
}
break;
default:{
//            NSLog(@"自动旋屏已锁");
}
break;
}
}

```

但是如果用户的自动旋屏开关是关闭的，那么我们就不能正确的得到屏幕旋转的通知。


***


## 利用CoreMotion.frameWork框架来检测屏幕方向

```
@property (nonatomic, strong) CMMotionManager* motionManager;
```


```

- (CMMotionManager *)motionManager {
if (_motionManager) {
return _motionManager;
}
_motionManager = [[CMMotionManager alloc] init];
_motionManager.deviceMotionUpdateInterval = 1/15.0;//设置更新频率
return _motionManager;
}
```

需要检测屏幕方向时

```

- (void)startGravityMotionUpdate {
if (self.motionManager.deviceMotionAvailable) {
[self.motionManager startDeviceMotionUpdatesToQueue:[NSOperationQueue currentQueue]
withHandler: ^(CMDeviceMotion *motion, NSError *error){
[self performSelectorOnMainThread:@selector(handleDeviceMotion:) withObject:motion waitUntilDone:YES];

}];
} else {
[self setMotionManager:nil];
}
}

```

回调方法

```

- (void)handleDeviceMotion:(CMDeviceMotion *)deviceMotion{
double x = deviceMotion.gravity.x;
double y = deviceMotion.gravity.y;
if (fabs(y) >= fabs(x)) {
if (y >= 0){
//            NSLog(@"poraitUpsidedown");
} else {
//            NSLog(@"portrait");
}
} else {
if (x >= 0){
//            NSLog(@"landscaperight");
} else{
//            NSLog(@"landscapeleft");
}
}
}
}

```

这样就可以实时得到屏幕的方向信息了，但是要求设备要有陀螺仪和重力感应装置。

***
