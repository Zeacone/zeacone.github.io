---
title: iOS绘制SVG之path
date: 2017-06-27 14:50:46
categories: iOS
tags: [iOS, SVG]

---

### SVG中的Path

#### 简介
SVG是一个矢量图片格式，用文本编辑器打开SVG文件的话，可以发现SVG其实是一个XML格式的文本文件，SVG的显示其实就是对XML内容进行解析然后绘制出来。由于SVG的内容较多，本文只讲解SVG中path的部分。

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


#### 移动命令M/m

M命令后面一般会跟着一个点的坐标，如M22,11，意思就是移动到点（22，11）的位置，path将以这个点为起点。

如果是M命令的话，由于是绝对定位，所以代码如下：

```objective-c
CGPoint p = CGPointMake(22,11);
[self.path moveToPoint:p];
```

如果是m命令的话，由于是相对定位，起点是上一path的终点，所以需要一个变量lastPoint来存储上一path的终点，所以m命令的代码如下：

```objective-c
// m命令的起点应该加上上一path的终点
CGPoint p = CGPointMake(self.lastPoint.x + 22, self.lastPoint.y + 11);
[self.path moveToPoint:p];
// 在移动完成后，需要把lastPoint修改为当前命令的终点
self.lastPoint = p;
```

#### 线段命令L/l

