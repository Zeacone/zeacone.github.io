---
title: 在Swift中实现深拷贝
date: 2017-05-24 11:02:44
tags: [Swift]
---

### 关于数组拷贝的例子

首先，先看一个例子。

``` swift
struct Teacher {
    var name: String
    var age: Int
}

class Student {
    var name: String
    var age: Int

    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

...

var teachers = [Teacher(name: "Alex", age: 32)]
var students = [Student(name: "Bob", age: 18)]

var teachersCopy = teachers
var studentsCopy = students

teachersCopy[0].name = "AlexCopy"
studentsCopy[0].name = "BobCopy"

print(teachers[0].name, teachersCopy[0].name) // "Alex AlexCopy"
print(students[0].name, studentsCopy[0].name) // "BobCopy BobCopy"
```

为什么会出现上述这种情况呢？根据 Apple 的文档来看：

> Each array has an independent value that includes the values of all of its elements. For simple types such as integers and other structures, this means that when you change a value in one array, the value of that element does not change in any copies of the array.

简单翻译一下的意思就是，每个数组包括它包含的所有元素都有其独立的值。对于简单类型例如整型和其它结构来说，这意味着当你改变一个数组中的值时，该元素的值在数组的其它副本中的值并不会随之改变。

> If the elements in an array are instances of a class, the semantics are the same, though they might appear different at first. In this case, the values stored in the array are references to objects that live outside the array. If you change a reference to an object in one array, only that array has a reference to the new object. However, if two arrays contain references to the same object, you can observe changes to that object’s properties from both arrays. 

但是，如果数组中的元素是类的实例的话，在这种情况下，数组中存储的只是存在于数组外部的对象的引用。如果在一个数组中更改对对象的引用，则只有该数组具有对新对象的引用。 但是，如果两个数组包含对同一对象的引用，则可以从两个数组中观察到对该对象属性的更改。

回到我们的例子，因为 `Teacher`是一个结构体，属于简单数据类型，当存放它的数组被 copy 的时候，同时复制了数组中所有元素的值。所以，更改`teachersCopy`中的元素的时候原数组不受影响。

而`Student`是一个`class`，当存放`Student`的数组被复制的时候，复制的其实是对象的引用，因此对`studentsCopy`中的元素更改值的时候，两个数组中的元素都被更改。

### 数组的深拷贝

那么如何实现包含类实例的深拷贝呢？众所周知，在`Objective-C`中自定义类要实现copy需要遵循`NSCopying`协议。在 Swift 中也是一样的做法。

首先，定义一个`Coping`协议。

``` swift
protocol Copying {
    func init(original: Self)
}

extension Copying {
    func copy() -> Self {
        return Self.init(original: self)
    }
}
```

然后，让我们的自定义类遵循这个协议，并实现相应的方法。同时，在 copy 数组的时候也要改变方式。

``` swift
class Student: Copying {
    required init(original: Student) {
        name = original.name
        age = original.age
    }

    ...
}

var studentsCopy = [Student]()
for student in students {
    let copiedStudent = student.copy()
    studentsCopy.append(copiedStudent)
}
```

这样，再修改`studentsCopy`中的内容，`students`就不会再随之改变了，数组的深拷贝也就实现了。

当然，还有更精简一些的写法。

```swift
extension Array where Element: Copying {
    func copy() -> Array {
        var copiedArray = Array<Element>()
        for element in self {
            copiedArray.append(element.copy)
        }
        return copiedArray
    }
}

...

var studentsCopy = students.copy()
```

That's done.