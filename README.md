##前言
最近在做个房地产的项目，需要有全景的效果。在网上google了一番，发现了[PanoramaGL](https://github.com/menssen/panoramagl)这个库，但是好久没更新了。尝试着拖到工程中用了下，更改了一些PLConstants.h的最大图片尺寸的参数，便可以跑起来了。

##但是
不幸的是，设置scrollEnable=YES后，滑动一下，便会一直不停的旋转。因为滑一下，在PLViewBase中，就会启动个timer/CADisplaylink进行刷新，不断调用drawViewInternally。后来想了个粗暴的办法，就是在drawViewInternally里面，延迟0.4s将其stop。但是这样一来，停的有点突然，不够平滑。╮(╯▽╰)╭，又不知道怎么去改源码，所以寻找另外的解决方法。

##新发现
还是google，发现有[pano2vr](http://ggnome.com/pano2vr)可以将全景图片转换成html，并且切换的很平顺。然后在app里面，弄个webview就够了。要吐槽一下的是，它是个付费软件，没找到mac的破解版，而mac的试用版，会打水印，没法用。后来还是切到了win版，下了个破解的。

##简单实用
用起来还是蛮方便的，将图片拖进去，支持一张全景图，或者6张立体的。然后设置参数，一般都不需要怎么设置，我只是将html设置了全屏。

还可设置hotspot，并且可以编辑皮肤，使用自定义的图片。总体还是比较方便的。[视频1](https://www.youtube.com/watch?v=1TRkos4Y-70) ，[视频2](https://www.youtube.com/watch?v=1TRkos4Y-70)

qq有一篇讲实现全景星球的文章，里面有比较详细的对各种全景工具的分析对比，[可戳这里](https://isux.tencent.com/3d.html)。

##接入到app
其实在这一步，还是有点坎坷的。主要是图片路径惹的祸。因为是以黄色文件夹加进去的，所以所有的资源都在mainbundle下面，没有层级了，而它自动生成的xml里面的image路径，是images/xx.jpg的。所以需要手动修改路径，将images/去掉，直接引用xx.jpg。

还有一点要注意，就是加载本地html时，baseURL的设定。需要设置为[[NSBundle mainBundle] bundleURL]。如果设置为nil，会出现提示。因为它找不js文件和img。

> This content requires HTML5/CSS3, WebGL, or Adobe Flash Player Version 9 or higher.

```
  NSString *path = [[NSBundle mainBundle] pathForResource:@"A5_cube" ofType:@"html"];
  NSString *data = [NSString stringWithContentsOfFile:path encoding:NSUTF8StringEncoding error:nil];

  [_webView loadHTMLString:data baseURL:[[NSBundle mainBundle] bundleURL]];
```

最后，如果还遇到js找不到或没有执行的情况，请检查build phase-->compile source里有没有它的身影。如有，将其拖至copy  bundle resource中。

不过我用xcode 7.3直接拖进去，是正常的。

呼哈哈，坑，终于填满了。

##6.25更新
=======
[Demo在此](https://github.com/silan-liu/Panorama.git)


##7.25更新
前段时间请教过一个前辈，关于自动停止滑动的问题。解决方案是设置一个摩擦因子(0-1)，在滑动是不断的乘以该因子，以达到减速的目的。简单的贴下代码吧。

在PLViewBase.m里面，drawViewInternally

```
-(void)drawViewInternally
{
	if(scene && !isValidForFov && !isSensorialRotationRunning)
    {
        PLCamera *camera = scene.currentCamera;

        [camera rotateWithVelocity:self.velocity];

        if (touchStatus==PLTouchEventTypeEnded) {
            //在PLCamera自定义一个转动方法，手指离开触摸屏的时候按画面按最后产生的速度及方向转动
            [camera rotateWithVelocity:self.velocity];
        }
        else {
            [camera rotateWithStartPoint:startPoint endPoint:endPoint];
        }

        if(delegate && [delegate respondsToSelector:@selector(view:didRotateCamera:rotation:)])
            [delegate view:self didRotateCamera:camera rotation:[camera getAbsoluteRotation]];
    }
    if(renderer)
        [renderer render];

    //利用摩擦因子减速
    if (touchStatus==PLTouchEventTypeEnded) {
        //摩擦因子
        float friction = 0.9;

        if (fabs(self.velocity.x)>1||fabs(self.velocity.y)>1) {

            self.velocity=CGPointMake(self.velocity.x*friction, self.velocity.y*friction);
        }
        else
        {
            [self stopAnimationInternally];
        }
    }
}
```

然后self.velocity需要自己计算，在move的时候根据touch移动的距离除以时间。

```
- (CGPoint)calculateVelocity:(NSSet *)touches {

    UITouch *touch = [touches anyObject];
    CGPoint location = [touch locationInView:self];
    CGPoint prevLocation = [touch previousLocationInView:self];

    CGFloat xDistance = location.x - prevLocation.x;
    CGFloat yDistance = location.y - prevLocation.y;

  // self.curTimestamp，self.previousTimestamp自己定义
    NSTimeInterval time = self.curTimestamp - self.previousTimestamp;
    if (time != 0) {
        CGFloat xSpeed = xDistance/time;
        CGFloat ySpeed = yDistance/time;

        return CGPointMake(xSpeed, ySpeed);
    }

    return CGPointZero;
}
```

最后在PLObject中添加rotateWithVelocity的方法

```
-(void)rotateWithVelocity:(CGPoint)velocity
{
    float sensitivity = 900;

    self.pitch += velocity.y/ sensitivity;
    self.yaw += -velocity.x/sensitivity;
}
```
