---
title: 【翻译】在Swift中实现MVVM
date: 2020-06-13 00:50:29
categories: iOS
tags: [iOS, Swift, MVVM]
---

原文链接：[How to Implement the MVVM Design Pattern in Swift](https://www.iosapptemplates.com/blog/ios-development/mvvm-swift)

如你所知，在 Swift 中有许多适用的设计模式比如 MVC，MVP 和 MVVM。我们根据项目的需求来选择最合适的。在iosapptemplates.com，我们使用了多种模式来构建我们的函数式App。在这片文章中，我们将指出使用MVVM所带来的一些优势并且用一些Swift代码片段进行清晰地解释。

MVVM代表Model - View（View 在此处表示ViewController）- ViewModel（VM）。那这是什么意思呢？这意味着我们将要引入一个新的叫做View Model的组件来帮助处理繁重的工作。

先前，所有事情包括网络，获取响应，处理来自UI的信号等等都是在View Controller中进行。这使得View Controller臃肿过载。所以在MVVM中，我们将把上述工作和代理职责划分到view model中去。

### 组件概览和具体职责
所以总的来说，我们有：
- View Controller：只展示和UI相关的事情——显示或者获取信息，View层的一部分
- View Model：接收来自VC的信息，处理所有接收到的信息并返回给VC
- Model：这只是model，同MVC中的model完全一样。VM使用它，并在VM发送新更新时进行更新

![](https://www.iosapptemplates.com/wp-content/uploads/2019/03/Screen-Shot-2019-03-17-at-8.25.44-PM.png)

在一个实际的项目中，应该记住两个主要的点：
1. 像这样分离工作所带来的便利。
2. 负责任何特定部分的每个组件都将彼此完全独立。

### MVVM Swift例子——构建一个登录页面
在这篇文章中，我们将实现一个在View Model中来处理数据的登录功能。对于这个比较，我将忽略自动布局的部分和xib的变化，集中在VC和VM的联系上。现在，让我们开始吧。

![](https://www.iosapptemplates.com/wp-content/uploads/2019/03/Screen-Shot-2019-03-17-at-8.29.18-PM.png)

``` swift
class ViewController: UIViewController {
    @IBOutlet weak var emailTextField: UITextField!
    @IBOutlet weak var passwordTextField: UITextField!
    @IBOutlet weak var loginButton: UIButton!

    override func viewDidLoad() {
        super.viewDidLoad()
    }

    @IBAction func didTapOnLoginButton(_ sender: Any) {

    }
}
```

这就是在这个启动项目中我所得到的一切。我们有所有的IBOutlet和IBAction——当然，还有两个重要的文件（VM，VC）。

我们将在这里讨论每个文件。如我们在文章开头所见，我们都知道VC的角色是只做展示和从UI获取信息。我们将在此处应用这些点：
- 从UI获取信息：输入框的输入信息
- 展示和响应来自VM的数据：点击登录按钮之后，我们将看见一条表示用户是否登录成功的消息。这取决于来自VM的信号

好了，问题就是我们如何将这两个文件彼此联系起来。有两种常用方法：protocol-delegate和RxSwift。在这一部分，我将谈谈protocol-delegate。这种方法有两个优点：
- 非常清楚明确，我们可以知道两个类之间传递的基本信息
- 负责任何特定部分的组件将彼此完全独立

#### 1. 从View Controller发送消息到View Model
这一部分里，有三个步骤：
##### 步骤1: 创建一个ViewModelDelagte的协议
这个协议是View Model的一部分，它有一个函数叫做：
`func sendValue(from emailTextField: String?, passwordTextField: String?)`

``` swift
protocol ViewModelDelegate {
    func sendValue(from emailTextField: String?, passwordTextField: String?)
}
```
##### 步骤2: 将View Model连接到视图层
接着，我们将在view controller中创建一个 viewModel 变量，从这里发送emailTextField和passwordTextField的值。
``` swift
var viewModel = ViewModel()

override func viewDidLoad() {
    super.viewDidLoad()
}

@IBAction func didTapOnLoginButton(_ sender: Any) {
    viewModel.sendValue(from: emailTextField.text, passwordTextField: passwordTextField.text)
}
```
好了，看起来挺酷，对吧？看起来我们已经将信息打包封装并发送出去了，接下来我们将看到在哪里去接收并解包这个信息。

##### 步骤3: 在View Model中处理Actions
在从view controller得到信息之后，我们可以在View Model像这样处理：
``` swift
func sendValue(from emailTextField: String?, passwordTextField: String?) {
    guard let emailTextField = emailTextField else { return }
    guard let passwordTextField = passwordTextField else { return }
    print("emailTextField: \(emailTextField)")
    print("passwordTextField: \(passwordTextField)")
}
```
运行并填充一些数据， 看看会发生什么。
![](https://www.iosapptemplates.com/wp-content/uploads/2019/03/Screen-Shot-2019-03-17-at-8.37.43-PM.png)

可以看到，我接收到了来自VC的“Email Testing”和“Password Testing”。

#### 2. 从view model发送数据返回到视图层
让我们先创建另一个叫做`ViewControllerDelegate`的协议，并让view controller遵循这个协议。这意味着我们不关心view model做了什么，我们只是需要操作的结果。

这是MVVM巨大的优点。负责VC的不会在乎VM所做的事情，他们只需要结果。同样地，VM不需要知道VC从哪里或如何从UI获取到值，它只关心需要对这些值做什么。因此，所有的关注点都是分离的，从而使所有组件彼此独立。

在我的例子中，你可以看到编写view model的人将关注的是需要对emailTextField和passwordTextField做的事情，但是不需要知道这些值是从何而来。这个业务逻辑归属于视图层。

``` swift
protocol ViewControllerDelegate {
    func getInformationBack(handledString: String?)
}
```
以上代码是前文所提到过的，它有一个方法名为`getInformationBack`。这意味着我们将通过此方法获取到VM所返回的值。

在ViewModel中，让我们添加一个weak类型的delegate变量：
``` swift
weak var delegate: ViewControllerDelegate?
```
并将从emailTextField和passwordTextField取回的值发送出去。在此篇教程中，我不会处理任何过于复杂的事情，仅仅只是将两个字符串合并为一，如下所示：
``` swift
func sendValue(from emailTextField: String?, passwordTextField: String?) {
    guard let emailTextField = emailTextField else { return }
    guard let passwordTextField = passwordTextField else { return }
    print("emailTextField: \(emailTextField)")
    print("passwordTextField: \(passwordTextField)")
    delegate?.getInformationBack(handledString: emailTextField + passwordTextField)
}
```
好的，到此，我们在处理了值（实际上我们仅仅合并了而已，但是在一个实际的项目中，你可能会做更多的事情比如验证邮箱、密码并告诉VC它们是否可用）之后封装了最后的输出。现在让我们看看如何将数据返回到view controller。

在VC的`viewDidLoad`里，添加一行代码：`viewModel.delegate = self`。还记得我们在VM里有一个delegate变量对吧？这行代码是VC告诉VM它将执行此委托。

然后，我们只需要实现协议并从VM取回数据。
``` swift
extension ViewController: ViewControllerDelegate {
    func getInformationBack(handledString: String?) {
        guard let handledString = handledString else { return }
        print("The string get from VM: \(handledString)")
    }
}
```
再次编译并运行Xcode项目。现在我们将拥有所需要的一切。我们可以将数据从VC发送到VM，反之亦然。

![](https://www.iosapptemplates.com/wp-content/uploads/2019/03/Screen-Shot-2019-03-17-at-8.52.33-PM.png)

### 结论

让我们总结一下我们所做的。在MVVM里，我们将与计算，处理数据等相关的任务分配给了VM。我们仅允许视图层执行UI琐事并从UI和用户交互中获取值。这样，每个设计组件仅负责一个专门的关注点，并且它们彼此完全独立。就这样。我希望在本文之后，您将对MVVM Swift以及如何使用protocol-delegate设计模式进行实现有一个概观。

MVVM是一个强力的Swift架构模式，它使程序员可以分离其设计组件的关注点，因此，对于任何出色的iOS工程师来说，掌握它都是至关重要的。