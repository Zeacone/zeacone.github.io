---
title: iOS中二维码的识别
date: 2017-09-16 10:10:21
categories: iOS
tags: [iOS, 二维码]
---



### 概览

> **二维条码**是指在一维条码的基础上扩展出另一维具有可读性的条码，使用黑白矩形图案表示二进制数据，被设备扫描后可获取其中所包含的信息。一维条码的宽度记载着数据，而其长度没有记载数据。二维条码的长度、宽度均记载着数据。
>
> 二维条码在商业活动中应用广泛，特别是在高科技行业、储存运输业、批发零售业等需要对物品进行廉价快捷的标示信息的行业用途广泛。

闲话少叙，二维码的应用已经到了无孔不入的地步，只要是个规模稍大点儿的APP就基本支持二维码的识别，所以，本文就讲讲二维码的识别。

<!-- more -->



### UI绘制

常规的二维码扫描界面是一个灰色的背景和一个正方形的白框来确定二维码的位置。在此我们使用贝塞尔曲线来画出形状，用`CAShapeLayer`来进行展示。

```swift
let path1 = UIBezierPath(rect: self.view.bounds)

// 第二条path用来绘制正方形
let x = (self.view.bounds.size.width - 200) / 2
let path2 = UIBezierPath(rect: CGRect(x: x, y: 150, width: 200, height: 200))

path1.append(path2)

let shape = CAShapeLayer()
shape.path = path1.cgPath
// 设置填充规则
shape.fillRule = kCAFillRuleEvenOdd
shape.fillColor = UIColor.lightGray.cgColor

self.view.layer.addSublayer(shape)
```
到此，扫描二维码的界面就算是构建完了。



### 摄像头扫描二维码

要使用摄像头扫描二维码，首先需要获取相机的授权。如果没有得到授权，应该提示用户在权限设置中打开相机的权限。



#### 检查相机授权

首先需要检查当前相机的授权状态，授权状态是一个枚举类型，根据不同的枚举值判断下一步应该要做的是什么。

- notDetermined，表示还没有发起过授权请求；

- authorized，表示已经取得了授权；

- denied，表示已经拒绝了授权；

- restricted，表示主动限制。



```swift
func checkCameraAuthorization() {
    let status = AVCaptureDevice.authorizationStatus(for: .video)

    switch status {
        case .notDetermined:
            AVCaptureDevice.requestAccess(for: .video, completionHandler: { (granted) in
                if granted {
                    // 如果拿到了相机授权，可以进行下一步
                    self.setupCamera()
                }
            })

        case .authorized:
            self.setupCamera()

        case .denied:
            print("前往系统设置打开app相机权限")
            // 此处需要在URL Types中添加一个prefs的URL Schemes
            UIApplication.shared.openURL(URL.init(string: "hello")!)

        case .restricted:
            print("Camera cannot be used.")

        default:
            break
    }
}
```


#### 相机设置

相机设置包括device、session、input和output的设置，其中最主要的是`rectOfInterest`的设置。`rectOfInterest`是一个`CGRect`类型的属性，但是它的坐标原点是在右上角，取值范围是0~1。也就是说右上角坐标为（0，0），左下角坐标是（1，1）。为了方便设置，此处谢了一个坐标转换的函数。

```swift
/// 将常规rect转换为相机的识别rect
///
/// - Parameters:
///   - rect: 待转换的常规rect
///   - parentSize: 所在的父视图的size
/// - Returns: 转换后的rect
func transaction(rect: CGRect, parentSize: CGSize) -> CGRect {
    let x = (parentSize.width - rect.origin.x) / parentSize.width
    let y = rect.origin.y / parentSize.height

    let width = rect.size.width / parentSize.width
    let height = rect.size.height / parentSize.height

    return CGRect(x: x, y: y, width: width, height: height)
}
```

接下来，检查back camera是否存在，设置session，input和output，最后设置显示相机捕获内容的`AVCaptureVideoPreviewLayer`并将其添加到父视图上去。最后调用`session`的`startRunning`方法开始捕捉。

```swift
func setupCamera() {
    let devices = AVCaptureDevice.devices(for: .video)
    for obj in devices {
        if obj.position == .back {
            self.device = obj
            break
        }
    }

    if self.device == nil {
        print("Unavailabel back camera.")
        return
    }


    let input = try! AVCaptureDeviceInput.init(device: self.device!)
    let output = AVCaptureMetadataOutput()

    output.setMetadataObjectsDelegate(self, queue: DispatchQueue.main)
  
    // 在此处获取UI上画的白框rect，然后通过上面所写的函数进行坐标转换
    let x = (self.view.bounds.size.width - 200) / 2
    let rect = CGRect(x: x, y: 150, width: 200, height: 200))
    output.rectOfInterest = self.transaction(rect: rect, parentSize: self.view.bounds.size)
  
    // 设置识别对象为二维码
    output.metadataObjectTypes = [.qr]

    self.session = AVCaptureSession()
    self.session?.sessionPreset = .high
    self.session?.addInput(input)
    self.session?.addOutput(output)

    let previewLayer = AVCaptureVideoPreviewLayer.init(session: self.session!)
    self.view.layer.insertSublayer(previewLayer, at: 0)
    self.session?.startRunning()
}
```



#### 输出代理

如果扫描到二维码，那么将会调用扫描成功的代理方法，通过获取扫描到的`metadataObjects`数组来获得最终的扫描文字内容。

```swift
func metadataOutput(_ output: AVCaptureMetadataOutput, didOutput metadataObjects: [AVMetadataObject], from connection: AVCaptureConnection) {
    let object: AVMetadataMachineReadableCodeObject = metadataObjects.first as! AVMetadataMachineReadableCodeObject
    print(object.stringValue as Any)
}
```



### 图片识别二维码

图片识别相当简单，识别二维码主要会用到`CoreImage`中的`CIDectector`类。

```swift
func scanQRCode(image: UIImage) {
    let detector = CIDetector.init(ofType: CIDetectorTypeQRCode, context: nil, options: [CIDetectorAccuracy: CIDetectorAccuracyHigh])

    let result = detector?.features(in: image.ciImage!)

    guard result else {
        print("Cannot detect QRCode on the image.")
        return
    }

    for feature in result {
        print("Message is \(feature.messageString).")
    }
}
```



### 使用闪光灯

在黑暗环境下，使用相机扫描二维码时，会需要有光源照明，比如夜晚扫摩拜、ofo之类的场景。此时就会需要打开闪光灯。

```    swift
func flashLight(sender: UIButton) {
    sender.isSelected = !sender.isSelected

    self.session?.beginConfiguration()
    self.device?.lockForConfiguration()

    self.device?.torchMode = sender.isSelected ? .on : .off
    self.device?.flashMode = sender.isSelected ? .on : .off

    self.device?.unlockForConfiguration()
    self.session?.commitConfiguration()
}
```

后续会补充根据光线强度自动打开闪光灯的部分。