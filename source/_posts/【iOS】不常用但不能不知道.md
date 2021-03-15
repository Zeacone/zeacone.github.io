---
title: 【iOS】不常用但不能不知道
date: 2018-03-15 10:20:12
tags: [iOS]
---

<!-- TOC -->

- [动态加载字体](#动态加载字体)

<!-- /TOC -->

### 动态加载字体
在一些阅读类的应用中，可能会有更换字体的需求。常规的字体更换是将字体文件添加到项目中，并在 `Info.plist` 文件中添加好参数。此处的字体添加方式为从服务器下载字体并添加，不用设置 `Info.plist`。
```swift
import CoreGraphics
import CoreText

...

func loadFont(_ fontName: String, size: CGFloat = 14) -> UIFont? {
        var result: UIFont?
        
        defer {
            return result
        }
        
        // 读取本地字体文件
        guard let path = Bundle.main.path(forResource: fontName, ofType: "ttf") else { return }
        guard let data = NSData(contentsOfFile: path), let fontData = data as? CFData else { return }

        // 通过 CGDataProvider 生成 CGFont 对象
        guard let fontDataProvider = CGDataProvider(data: fontData) else { return }
        guard let fontRef = CGFont(fontDataProvider) else { return }

        // 注册字体
        var error = Unmanaged<CFError>?(nilLiteral: ())
        CTFontManagerRegisterGraphicsFont(fontRef, &error)

        // 获取字体的实际名称
        guard let realFontName = fontRef.fullName as? String else { return }
        
        guard let font = UIFont(name: realFontName, size: size) else { return }
        result = font
    }
```