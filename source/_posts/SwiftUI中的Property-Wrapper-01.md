---
title: 'SwiftUI中的Property Wrapper #01'
date: 2020-02-12 10:34:36
tags:
---

在学习 `SwiftUI` 的过程中，出现了许多在 `Swift` 中未曾遇到过的类似 `@xxx` 的东西，在 `SwiftUI` 中被称作 `Property Wrapper`。今天就来学习一下 `Property Wrapper` 应该怎么使用以及应该什么时候使用。

`SwiftUI` 中常见的 `Property Wrapper` 有 `@State`，`@Binding`，`@ObservedObject`，`@EnvironmentObject` 以及 `@Environment`等等。接下来就开始分别学习。

### @State
`@State` 用来描述视图的状态，它所修饰的变量被存储在视图结构之外一个特殊的内部内存空间，只有相关联的视图能够访问。一旦 `@State` 所修饰的变量发生改变，`SwiftUI` 将根据所发生的改变重建试图。下面是一个例子：
```swift
struct ProductsView: View {
    let products: [Product]
    
    @State private var showFavorited: Bool = false
    
    var body: some View {
        List {
            Button(
                action: { self.showFavorited.toggle() },
                label: { Text("Change filter")}
            )
            
            ForEach(products) { product in
                if !self.showFavorited || product.isFavorited {
                    Text(product.title)
                }
            }
        }
    }
}
```

在上面的例子中，一旦按下按钮，`@State` 所修饰的`showFavorited` 的值就发生改变，整个视图也将被重新创建。

### @Binding
`@Binding` 为值类型提供了引用访问的方式。有时我们需要使子视图能够访问父视图的状态，但是不能够简单地把值传递过去，因为状态是值类型，SwiftUI 只会传递这个值的一个拷贝。这时候 @Binding 就派上用场了：
```swift
struct FilterView: View {
    @Binding var showFavorited: Bool
    
    var body: some View {
        Toggle(isOn: $showFavorited) {
            Text("Change filter")
        }
    }
}

struct ProductsView: View {
    let products: [Product]
    
    @State private var showFavorited: Bool = false
    
    var body: some View {
        List {
            FilterView(showFavorited: $showFavorited)
            
            ForEach(products) { product in
                if !self.showFavorited || product.isFavorited {
                    Text(product.title)
                }
            }
        }
    }
}
```

在上面的例子中，使用 @Binding 去标记 showFavorited ，这样的话，FilterView 对showFavorited的读写实际就是对ProductsView中的showFavorited进行读写。而根据上一节 @State 所讲到的内容，ProductsView 的 showFavorited 一改变，SwiftUI 就将重新创建此视图。

> @Binding只适合对值类型的变量进行修饰，如果对非值类型的变量进行修饰可能会引起未知的错误。
>
> 另外，对 @Binding 修饰的变量传值的时候一定要加上 `$` 符号，代表是传递的引用，否则就是传递值的拷贝。

### @ObservedObject
@ObservedObject 是 Combine 框架中的一部分，主要用于处理 SwiftUI 之外的一些事物——比如业务逻辑。可以在不同的独立的视图之间共享、观察和订阅 @ObservedObject 所修饰的变量。一旦变量发生改变，所有绑定此变量的视图都将重建。
```swift
import Combine

final class PodcastPlayer: ObservableObject {
    @Published private(set) var isPlaying: Bool = false
    
    func play() {
        isPlaying = true
    }
    
    func pause() {
        isPlaying = false
    }
}
```

`PodcastPlayer` 可以在多个视图之间共享，在 `@Published` 属性的帮助下，`SwiftUI` 可以追踪 `@ObservableObject` 的变化。一旦 `@Published` 发生变化，所有绑定 `@ObservableObject` 的视图都将被重建。

```swift
struct EpisodesView: View {
    @ObservedObject var player: PodcastPlayer
    let episodes: [Episode]
    
    var body: some View {
        List {
            Button(
                action: {
                    if self.player.isPlaying {
                        self.player.pause()
                    } else {
                        self.player.play()
                    }
                },
                label: { Text(player.isPlaying ? "Pause": "Play") }
            )
            ForEach(episodes) { episode in
                Text(episode.title)
            }
        }
    }
}
```

> 因为可以在多个视图之间共享数据，因此 ObservableObject 必须是引用类型。

### @EnvironmentObject
@EnvironmentObject 可以将变量隐式地注入到试图层级的环境中。举例说明：
```swift
...
let player = PodcastPlayer()
window.rootViewController = UIHostingController(
    rootView: EpisodesView(episode: episodes)
        .environmentObject(player)
)
...

struct EpisodeView: View {
@EnvironmentObject var player: PodcastPlayer
    ...
}
```

如上可见，我们可以通过 `environmentObject` 修饰符来传递 `PodcastPlayer`，而且可以通过 `@EnvironmentObject` 轻松地访问。`@EnvironmentObject` 通过动态成员查找（dynamic member lookup）特性去找到 `Environment` 中的 `PodcastPlayer` 实例。这就是为什么不需要在 `EpisodesView` 初始化的时候传递 `PodcastPlayer` 实例的原因。

### @Environment
前一章节讨论的是将自定义对象传递到 `Environment` 中，但是 `SwiftUI` 已经存在许多系统级的设置，通过 `@Environment` 可以轻松地进行访问。

```swift
struct CalendarView: View {
    @Environment(\.calendar) var calendar: Calendar
    @Environment(|.locale) var locale: Locale
    @Environment(\/colorScheme) var colorScheme: ColorScheme
    
    var body: some View {
        return Text(locale.identifier)
    }
}
```

通过 `@Environment` 实现了访问和订阅系统级设置的变化。这些系统设置变化，`App` 中对应的视图也将被重建。