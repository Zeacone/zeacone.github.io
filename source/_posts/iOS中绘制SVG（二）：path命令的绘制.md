---
title: iOS中绘制SVG（二）：path命令的绘制
date: 2017-08-26 12:59:28
categories: iOS
tags: [iOS, SVG]
---

Path 命令的绘制分为两个部分，第一步是解析 path 的命令和参数，第二部是根据解析出的数据生成正确的贝塞尔曲线然后进行绘制。

Path 命令一共有 10 个子命令，并且区分大小写，大写代表绝对定位，小写代表相对定位，为了方便描述，在下文所有命令都以大写表示，只在代码中进行大小写——即相对定位和绝对定位的区分。

- M：移动命令，即 move；
- L：线段命令，即 line；
- H：根据上一条 path 的终点添加一条水平方向的线段；
- V：根据上一条 path 的终点添加一条垂直方向的线段；
- A：添加一段弧线；
- C：添加一段 curve，即二次贝塞尔曲线；
- Q：添加一段 quad curve，即三次贝塞尔曲线；
- S：在 C 命令后追加一段镜像 curve；
- T：在 Q 命令后追加一段镜像 quad curve；
- Z：close 当前的path。

<!-- more -->

接下来就开始 path 命令具体的解析和绘制过程。



### 解析数据



#### 扫描命令和参数

首先，path 命令以一个字符串的形式表示，因为在 path 中都是以 command 加上参数的形式，所以在此需要使用到`NSScanner`这个类来进行扫描相应的命令和参数。

先将所有的命令分大小写拼接成一个字符串——“MLHVACQSTZmlhvacqstz”，然后利用`scanCharactersFromSet:`方法扫描子命令，如果扫描到命令的话继续利用`scanDouble:`扫描命令后面所跟的参数，参数扫描结束后开始生成本次命令的贝塞尔曲线。
```objc
NSScanner *scanner = [NSScanner scannerWithString:commandStr];
NSCharacterSet *skipSet = [NSCharacterSet characterSetWithCharactersInString:@" ,\n"];
scanner.charactersToBeSkipped = skipSet;

NSCharacterSet *commandSet = [NSCharacterSet characterSetWithCharactersInString:@"MLACQSTHVZmlacqsthvz"];
NSString *command = nil;

while ([scanner scanCharactersFromSet:commandSet intoString:&command]) {
    NSMutableArray<NSNumber *> *numbers = [NSMutableArray array];
    double number = 0;
        
    while ([scanner scanDouble:&number]) {
        [numbers addObject:@(number)];
    }
    [self executeCommand:[command UTF8String] Numbers:[numbers copy]];
}
```


#### 执行对应的命令

然后在`executeCommand:(const char *)command Numbers:(NSArray<NSNumber *> *)nums`方法中根据具体命令的大小写来执行对应的方法。


```objc
- (void)executeCommand:(const char *)command Numbers:(NSArray<NSNumber *> *)nums {

  switch (command[0]) {
      case 'M':
          [self move:nums Relative:NO];
          break;
      case 'm':
          [self move:nums Relative:YES];
          break;
      case 'L':
          [self addLine:nums Relative:NO];
          break;
      case 'l':
          [self addLine:nums Relative:YES];
          break;
      case 'A':
          [self addArc:nums Relative:NO];
          break;
      case 'a':
          [self addArc:nums Relative:YES];
          break;
      case 'C':
          [self addCurve:nums Relative:NO];
          break;
      case 'c':
          [self addCurve:nums Relative:YES];
          break;
      case 'Q':
          [self addQuad:nums Relative:NO];
          break;
      case 'q':
          [self addQuad:nums Relative:YES];
          break;
      case 'S':
          [self addSmoothCurve:nums Relative:NO];
          break;
      case 's':
          [self addSmoothCurve:nums Relative:YES];
          break;
      case 'T':
          [self addSmoothQuad:nums Relative:NO];
          break;
      case 't':
          [self addSmoothQuad:nums Relative:YES];
          break;
      case 'H':
          [self addHorizon:nums Relative:NO];
          break;
      case 'h':
          [self addHorizon:nums Relative:YES];
          break;
      case 'V':
          [self addVertical:nums Relative:NO];
          break;
      case 'v':
          [self addVertical:nums Relative:YES];
          break;
      case 'Z':
      case 'z':
          [self close];
          break;

      default:
          NSLog(@"cannot resolve command.");
          break;
  }
}
```
到此为止，path 的解析就已经完成了。



### 绘制path

由于需要保存绘制过程中的一些变量，因此会声明几个全局变量在绘制过程中进行保存数据。

- 由于存在相对位置，所以需要`CGPoint`类型的`lastPoint`来保存前一path的终点；
- 由于T命令需要根据上一条路径推断控制点，所以需要`CGPoint`类型的`preQCtrl`来记录上一条二次贝塞尔曲线的控制点；
- 由于 S 命令同T命令一样，所以需要`CGPoint`类型的`preCCtrl`来记录上一条三次贝塞尔曲线的第一个控制点；

```objc
@property (nonatomic, assign) CGPoint lastPoint;
@property (nonatomic, assign) CGPoint preQCtrl;
@property (nonatomic, assign) CGPoint preCCtrl;
```


#### M命令

M 命令后面会跟着一个坐标，意思是将画笔移动到所给坐标的位置，path 将以这个坐标为起点。

```objc
// M命令格式如下："M10 10"
CGPoint p = CGPointMake(10, 10);

// 如果是相对定位，需要加上上一 path 的终点
if (relative) {
    p = CGPointMake(lastPoint.x + p.x, lastPoint.y + p.y);
}
[self.path moveToPoint:p];

// 在绘制完成后，需要将记录 path 终点的 lastPoint 指向新的终点，后面除了 Z 以外的所有命令都相同，不再赘述。
lastPoint = p;
```


#### L命令

L 命令将会在上一 path 的终点和给定的点之间画一条直线。L 命令添加一条直线，其后同样是跟着一个点的坐标，这个坐标点是直线的终点。

```objc
// L命令格式如下："L 10 10"
CGPoint p = CGPointMake(10, 10);

if (relative) {
    p = CGPointMake(lastPoint.x + p.x, lastPoint.y + p.y);
}

[self.path addLineToPoint:p];

lastPoint = p;
```


#### H命令

H 代表着 horizon，意思是画一条水平方向的直线。H 命令后面会跟一个数字，数字代表 x 坐标，y 坐标与前一坐标的 y 坐标相同。

```objc
// H命令格式如下："H 90"
CGPoint p = CGPointMake(90, lastPoint.y);

if (relative) {
    p = CGPointMake(lastPoint.x + p.x, lastPoint.y);
}
[self.path addLineToPoint:p];

lastPoint = p;
```


#### V命令

V 代表 vertical，意思是画一条垂直方向的直线。V 命令格式同 H 命令一样。

```objc
// V命令格式如下："V 90"
CGPoint p = CGPointMake(lastPoint.x, 90);

if (relative) {
    p = CGPointMake(lastPoint.x, lastPoint.y + p.y);
}
[self.path addLineToPoint:p];

lastPoint = p;
```



#### Q命令
Q 命令用来创建二次贝塞尔曲线，关于贝塞尔曲线，可以去[维基](https://zh.wikipedia.org/wiki/%E8%B2%9D%E8%8C%B2%E6%9B%B2%E7%B7%9A#.E4.BA.8C.E6.AC.A1.E6.96.B9.E8.B2.9D.E8.8C.B2.E6.9B.B2.E7.B7.9A)上查看其具体的定义。二次贝塞尔曲线需要两个点来确定，分别是终点和控制点。iOS 中的`UIbezierPath`类提供了`addQuadCurveToPoint: controlPoint:`方法来绘制一条二次贝塞尔曲线。

由于相对定位是相对于前一 path 的终点来说，而二次贝塞尔曲线因为有两个点需要在本次绘制的时候确定，因此，绘制的时候需要同时计算控制点和终点的坐标。

```objc
// Q命令格式如下："Q 95 10 180 80"，其中(95, 10)是控制点坐标，(180, 80)是终点坐标
CGPoint c = CGPointMake(95, 10); // 控制点
CGPoint p = CGPointMake(180, 80); // 终点

if (relative) {
    c = CGPointMake(lastPoint.x + c.x, lastPoint.y + c.y);
    p = CGPointMake(lastPoint.x + p.x, lastPoint.y + p.y);
}

[self.path addQuadCurveToPoint:p controlPoint:c];

lastPoint = p;
// 因为下文中的 T 命令，所以需要保存二次贝塞尔曲线的控制点
preQCtrl = c;
```



#### T命令

T 命令实际上也是绘制一条二次贝塞尔曲线，与 Q 命令不同的是，T 命令不需要指定控制点坐标，只需要一个终点坐标。控制点坐标是前一段 path 的控制点相对于终点的对称点。如下图，蓝色直线与红色直线的交点为前一 path 的终点 p，红色直线向上延长的终点为前一 path 的控制点 c1 ，蓝色直线向下延长的终点为当前绘制曲线的控制点 c2 ，c1 和 c2 相对于 p 对称。因此 c2 的坐标计算公式为 `c2.x = p.x + (p.x - c1.x)，c2.y = p.y + (p.y - c1.y)`。
![](https://developer.mozilla.org/@api/deki/files/364/=Shortcut_Quadratic_Bezier.png)
___需要注意的是，T 命令前面必须也是一条二次贝塞尔曲线，即前一个命令必须是 Q 命令或者 T 命令。如果 T 命令单独出现的话，将会画出一条直线。___

```objc
// T命令格式如下："T95 10"，(95, 10)是二次贝塞尔曲线的终点
CGPoint p = CGPointMake(95, 10);
CGPoint c;

if (relative) {
    p = CGPointMake(lastPoint.x + p.x, lastPoint.y + p.y);
}

// 如果 T 命令前不是二次贝塞尔曲线，将会画出一条直线，也就是说控制点在终点和起点的连线上
if (CGPointEqualToPoint(preQCtrl, CGPointZero)) {
    c = p;
} else {
    c = CGPointMake(lastPoint.x + (lastPoint.x - preQCtrl.x), lastPoint.y + (lastPoint.y - preQCtrl.y));
}

[self.path addQuadCurveToPoint:p controlPoint:c];

lastPoint = p;
preQCtrl = c;
```



#### C命令

C 命令用来创建三次贝塞尔曲线。三次贝塞尔曲线需要三个点来确定，分别是终点和两个控制点。iOS 中的`UIbezierPath`类同样提供了`addCurveToPoint: controlPoint1: controlPoint2:`方法来绘制一条三次贝塞尔曲线。

```objc
// C命令格式如下："C 20 20, 40 20, 50 10"
// 控制点1：(20, 20)，控制点2：(40, 20)，终点：(50, 10)
CGPoint c1 = CGPointMake(20, 20);
CGPoint c2 = CGPointMake(40, 20);
CGPoint p  = CGPointMake(50, 10);

// 同Q命令一样，相对定位情况下需要计算所有控制点和终点的坐标
if (relative) {
  c1 = CGPointMake(lastPoint.x + c1.x, lastPoint.y + c1.y);
  c2 = CGPointMake(lastPoint.x + c2.x, lastPoint.y + c2.y);
  p = CGPointMake(lastPoint.x + p.x, lastPoint.y + p.y);
}

[self.path addCurveToPoint:p controlPoint1:c1 controlPoint2:c2];
lastPoint = p;
// 因为下文中的 S 命令，所以需要保存三次贝塞尔曲线的第二个控制点
preCCtrl = c2;
```


#### S命令

S 命令可以用来创建与之前 C 或者 S 命令一样的三次贝塞尔曲线。S 命令的第一个控制点是前一条三次贝塞尔曲线的第二个控制点相对于终点的对称点。如下图，蓝色和红色线段的交点为上一线段的终点 p，红色线段向上延长的终点为第一条线段的第二个控制点 c12，蓝色线段向下延长的终点为第二条线段的第一个控制点 c21，c12 和 c21 相对于 p 对称。因此 c21 的坐标计算公式为：`c21.x = p.x + (p.x - c12.x), c21.y = p.y + (p.y - c12.y) `。

![](https://developer.mozilla.org/@api/deki/files/363/=ShortCut_Cubic_Bezier.png)
___如果 S 命令单独使用，那么它的两个控制点将会使用同一个坐标。___

```objc
// S命令的格式如下："S 150 150, 180 80"
// 第二个控制点：(150, 150)，终点：(180, 80)
CGPoint c1;
CGPoint c2 = CGPointMake(150, 150);
CGPoint p = CGPointMake(180, 80);

if (relative) {
  c2 = CGPointMake(lastPoint.x + c2.x, lastPoint.y + c2.y);
  p = CGPointMake(lastPoint.x + p.x, lastPoint.y + p.y);
}

// 如果前一个命令不是C命令或者S命令，则第一个控制点与第二个控制点坐标相同
if (CGPointEqualToPoint(preCCtrl, CGPointZero)) {
  c1 = c2;
} else {
  c1 = CGPointMake(lastPoint.x + (lastPoint.x - preCCtrl.x), lastPoint.y + (lastPoint.y - preCCtrl.y));
}

[self.path addCurveToPoint:p controlPoint1:c1 controlPoint2:c2];

lastPoint = p;
preCCtrl = c2;
```



#### A命令

暂略。



#### Z命令 

Z命令代表封闭 path。

```objc
[self.path close];
```



#### 查漏
因为二次贝塞尔曲线和三次贝塞尔曲线都需要记录一个控制点，并且二次和三次贝塞尔曲线的记录不一样，所以在执行其它命令的时候，需要将记录清空（设置成CGPointZero）以免 T 和 S 命令误以为前一 path 是二次或者三次贝塞尔曲线。
因此在执行命令的`executeCommand:(const char *)command Numbers:(NSArray<NSNumber *> *)nums`方法开头部分添加如下代码：
```objc
// 如果命令不是 Q 和 T，需要将记录二次贝塞尔曲线控制点的属性 preQCtrl 设为 CGPointZero
// 因此在非 Q 和 T 命令的情况下，便于单独的 T 命令进行判断
NSString *q_excluded = @"MmLlAaCcSsHhVvZz";
if ([q_excluded containsString:[NSString stringWithUTF8String:command]]) {
    preQCtrl = CGPointZero;
}

// 如果命令不是 C 和 S，需要将记录三次贝塞尔曲线第二个控制点的属性 preCCtrl 设为 CGPointZero
// 因此在非 C 和 S 命令的情况下，便于单独的 S 命令进行判断
NSString *c_excluded = @"MmLlAaQqTtHhVvZz";
if ([c_excluded containsString:[NSString stringWithUTF8String:command]]) {
    preCCtrl = CGPointZero;
}
```

### 小结

本文只是简单介绍了 SVG 中 path 的基础绘制，其中主要的内容在于：

1. 将命令和参数正确的解析出来；

2. 弄清楚相对坐标和绝对坐标情况下的 path 的点所应处的位置；

3. 另外就是弄清楚T和S命令的控制点是怎么确定的。

其它方面的东西都是很容易的。

详细代码请关注GitHub：[Zeacone](https://github.com/Zeacone/SVGlib)