---
title: iOS中绘制SVG（二）：path命令的绘制
date: 2017-08-26 12:59:28
categories: iOS
tags: [iOS, SVG]
---

#### Path命令介绍
Path命令共有10个，其中小写代表相对定位——path的起点即上一path的终点，大写代表绝对定位——path中的点都以坐标系原点为参考点。所有的path命令如下：

+ M/m：移动到某个点，如“M5， 5”的意思就是将path移动到坐标为(5， 5)的位置；
+ L/l：添加一条直线到某一个点，如“L5, 5”；
+ A/a：添加一条二次弧线；
+ C/c：添加一段curve；
+ Q/q：添加quad curve；
+ S/s：在c命令后面追加一段镜像的curve；
+ T/t：在q命令后面追加一段镜像的quad curve；
+ H/h：添加一条横向的直线；
+ V/v：添加一条纵向的直线；
+ Z/z：close当前的path。

<!-- more -->


#### 移动命令M/m

M命令后面一般会跟着一个点的坐标，如<path d="M22 11"/>，意思就是将画笔移动到点（22，11）的位置，path将以这个点为起点。

如果是M命令的话，由于是绝对定位，所以代码如下：

```objc
CGPoint p = CGPointMake(22,11);
[self.path moveToPoint:p];
```

如果是m命令的话，由于是相对定位，起点是上一path的终点，所以需要一个变量lastPoint来存储上一path的终点，所以m命令的代码如下：

```objc
// m命令的起点应该加上上一path的终点
CGPoint p = CGPointMake(self.lastPoint.x + 22, self.lastPoint.y + 11);
[self.path moveToPoint:p];
// 在移动完成后，需要把lastPoint修改为当前命令的终点
self.lastPoint = p;
```

#### 线段绘制命令

Path的命令中一共包含3个，分别是L、V和H。

##### L/l命令

L命令将会在当前位置和新位置（L前面画笔所在的点）之间画一条线段。L命令添加一条线段，其后同样是跟着一个点的坐标，这个坐标点是line的终点，L命令的代码如下：

```objc
// 大写L命令
[self.path addLineToPoint:p];
```

l命令的代码如下：

```objc
CGPoint p1 = p;
p1 = CGPointMake(p1.x + lastPoint.x, p1.y + lastPoint.y);
[self.path addLineToPoint:p];
lastPoint = p1;
```

H和V命令：

H和V后面都只跟一个数字，分别代表目标坐标的x值和y值，新的坐标的y值和x值与lastPoint相同。

```objc
// 示例SVG path命令：d="H10"
CGPoint ph = CGPointMake(10, lastPoint.y);
[self.path addLineToPoint:ph];
// 示例SVG path命令：d="V20"
CGPoint pv = CGPointMake(lastPoint.x, 20)
[self.path addLineToPoint:pv];
```

相对定位的h和v命令如下：

```objc
// h命令
CGPoint ph = CGPointMake(10, lastPoint.y);
ph = CGPointMake(ph.x + lastPoint.x, lastPoint.y);
[self.path addLineToPoint:ph];
lastPoint = ph;
// v命令
CGPoint pv = CGPointMake(lastPoint.x, 20);
pv = CGPointMake(lastPoint.x, py.y + lastPoint.y);
[self.path addLineToPoint:pv];
lastPoint = pv;
```

#### C命令和S命令 



#### Q命令和T命令 

#### Z命令
