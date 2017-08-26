---
title: iOS中绘制SVG（一）：SVG文件的解析
date: 2017-08-24 18:58:16
categories: iOS
tags: [iOS, Objective-C, SVG]
---

### 0x01 SVG 的实质
SVG实质上是一个XML格式的文本文件，可以同普通的XML文件一样通过一些XML解析库来进行解析。因此绘制SVG的过程实际上是先解析XML文件，然后根据解析出来的命令在屏幕上绘制出来。

SVG的命令常见的有形状、transform、文字、渐变和滤镜。在本系列的文章中，只讨论基本的形状绘制，不涉及文字、渐变和滤镜的部分。

<!-- more -->

而形状部分主要包括7个类型，分别是：
+ path
+ line
+ circle
+ rect
+ ellipse
+ polyline
+ polygon

transform主要包括：
+ move
+ rotate
+ scale

而本文章主要就是讨论上述所说的命令本身和其拥有属性的解析。

### 0x02 SVG 元素的解析

第一步，先获取SVG的路径，使用iOS系统自带的XML解析工具进行解析。

```objc
NSData *data = [NSData dataWithContentsOfFile:fileName];
NSXMLParser *parser = [[NSXMLParser alloc] initWithData:data];
parser.delegate = self;
[parser parse];
```

第二步，实现XML解析的代理方法，通过`elementName`来创建对应形状的类。因为解析出来的形状拥有许多相同的属性，因此新建一个基类`SVGElement`，其它所有的形状都会继承此基类。基类包含的属性和方法如下：
```objc
@interface SVGElement : NSObject

- (instancetype)initWithAttribute:(NSDictionary *)attr;

@property (nonatomic, copy) NSString *title;
// 标识一个元素的id
@property (nonatomic, copy) NSString *identifier;
@property (nonatomic, copy) NSString *className;
// 该元素的transform属性字符串
@property (nonatomic, copy) NSString *tranform;
// 该元素所属的分组
@property (nonatomic, copy) NSString *group;

// 用来绘制该元素的贝塞尔曲线
@property (nonatomic, strong) UIBezierPath *path;

/**
*  绘制该元素时的stroke color和fill color
*/
@property (nonatomic, strong) UIColor *strokeColor;
@property (nonatomic, strong) UIColor *fillColor;


/**
*  标记此元素被绘制到屏幕上之后是否能够响应点击事件
*/
@property (nonatomic, assign) BOOL selectable;

@end
```
然后在XML解析的代理方法中实现对应形状的解析，通过`elementName`拼接成对应的类名，然后使用`NSClassFromString`生成对应的类。另外新建一个`elements`的数组用来存储所有解析出来的元素。
```objc
- (void)parser:(NSXMLParser *)parser didStartElement:(NSString *)elementName namespaceURI:(NSString *)namespaceURI qualifiedName:(NSString *)qName attributes:(NSDictionary<NSString *,NSString *> *)attributeDict {
    
    NSArray *names = @[@"path", @"rect", @"circle", @"ellipse", @"line", @"polyline", @"polygon"];
    
    // 获取SVG的共有属性，比如SVG的size等等
    if ([elementName isEqualToString:@"svg"]) {
        // 获取公有属性 like viewbox
        CGFloat width = attributeDict[@"width"].doubleValue;
        CGFloat height = attributeDict[@"height"].doubleValue;
        self.svgSize = CGSizeMake(width, height);
        
    } else if ([elementName isEqualToString:@"g"]) {
        // group
        self.transform = [attributeDict objectForKey:@"transform"];
    } else if ([names containsObject:elementName]) {
    // 根据元素的name生成对应的类，比如SVGCircle、SVGPath等
        NSString *className = [@"SVG" stringByAppendingString:[elementName capitalizedString]];
        
        Class myClass = NSClassFromString(className);
        
        SVGElement *element = [((SVGElement *)[myClass alloc]) initWithAttribute:attributeDict];
        
        if (element) {
        // 如果元素存在，获取对应的属性字符串
            if (!element.tranform && self.transform) {
                element.tranform = self.transform;
            }
            // 将解析出来的元素添加到数组中，便于解析完成之后的绘制
            [self.elements addObject:element];
        }
    }
}
```
在代理方法中，通过初始化方法`initWithAttribute:`已经将`attributeDict`属性拿到，会在初始化方法里面进行解析拆分出各个详细的属性，此处不进行详细的介绍，具体可看文章末尾所附上的GitHub链接。

### 0x03 SVG transform的解析
一个常见的transform属性字符串如下所示：
transform="translate(398.000000, 1925.000000) rotate(90.000000) translate(-398.000000, -1925.000000) translate(-1527.000000, 1527.000000)"

举例所示的transform包含了translate和rotate，另外常见的还有scale。此处要做的便是将transform字符串解析成iOS能识别的`CGAffineTransform`类型。

根据`translate`、`scale`和`rotate`这3个关键字利用正则表达式将transform字符串拆分成一个包含`NSTextCheckingResult`对象的数组，此时的`NSTextCheckingResult`对象实际上就是一个命令加上对应的参数，如`translate(398.000000, 1925.000000)`，然后通过`NSScanner`忽略掉某些特殊字符，将后面的参数扫描出来，就满足了构成一个`CGAffineTransform`对象的条件。按照transform字符串的顺序依次解析就构成了最终的`CGAffineTransform`。
需要注意的是，SVG中`rotate`使用的是角度，在iOS中需要转换成弧度再进行使用。

详细的转换代码如下：

```objc
+ (CGAffineTransform)transformFromString:(NSString *)transformString {
    CGAffineTransform transform = CGAffineTransformIdentity;
    
    NSError *error = nil;
    NSRegularExpression *regex = [NSRegularExpression regularExpressionWithPattern:@"translate|scale|rotate" options:NSRegularExpressionCaseInsensitive error:&error];
    
    NSArray<NSTextCheckingResult *> *checkResults = [regex matchesInString:transformString options:0 range:NSMakeRange(0, [transformString length])];
    
    NSScanner *scanner = [NSScanner scannerWithString:transformString];
    
    NSMutableCharacterSet *skippedCharacterSet = [[NSMutableCharacterSet alloc] init];
    [skippedCharacterSet formUnionWithCharacterSet:[NSCharacterSet letterCharacterSet]];
    [skippedCharacterSet formUnionWithCharacterSet:[NSCharacterSet characterSetWithCharactersInString:@",() "]];
    
    scanner.charactersToBeSkipped = skippedCharacterSet;
    
    for (NSTextCheckingResult *result in checkResults) {
        
        if ([[transformString substringWithRange:result.range] isEqualToString:@"translate"]) {
            CGFloat valueX;
            CGFloat valueY;
            [scanner scanDouble:&valueX];
            [scanner scanDouble:&valueY];
            transform = CGAffineTransformTranslate(transform, valueX, valueY);
        } else if ([[transformString substringWithRange:result.range] isEqualToString:@"rotate"]) {
            CGFloat angle;
            [scanner scanDouble:&angle];
            
            transform = CGAffineTransformRotate(transform, [self.class radianFromAngle:angle]);
            
        } else if ([[transformString substringWithRange:result.range] isEqualToString:@"scale"]) {
            
            CGFloat scale;
            [scanner scanDouble:&scale];
            transform = CGAffineTransformScale(transform, scale, scale);
        }
    }
    
    return transform;
}

+ (CGFloat)radianFromAngle:(CGFloat)angle {
    return angle / 180.0 * M_PI;
}
```


### 0x04 SVG Path命令的解析
Path命令的解析由于和path的绘制关联紧密，因此会在path的绘制章节中一起进行讲解。

关于SVG的解析绘制所有源码包括demo可在GitHub进行查看：[SVGlib](https://github.com/Zeacone/SVGlib)