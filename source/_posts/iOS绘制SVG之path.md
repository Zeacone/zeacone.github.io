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
+ M、m：移动到某个点，如“M5， 5”的意思就是将path移动到坐标为(5， 5)的位置；
+ L、l：添加一条直线到某一个点，如“L5, 5”；
+ A、a：添加一条二次弧线；
+ C、c：添加一段curve
+ Q、q：添加quad curve
+ S、s：
+ T、t
+ H、h
+ V、v
+ Z、z