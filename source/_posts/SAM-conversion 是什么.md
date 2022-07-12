## 关于 SAM-conversion

看到一些注释提到 SAM-conversion，查找之后记录下。



SAM全称是 Single Abstract Method，当 Interface 只有一个抽象方法时，它可以当作一个函数来调用。比如View点击接口：

```java
// 单一抽象方法的接口
inteface OnClickListener{
    onClick(View v);
}

class View{
    // 形参需要一个接口实例
    void setOnClickListener(OnClickListener l) {
        
    }
}

// 不使用lambda，显式的创建实例
view.setOnClickListener(object:View.OnClickListener{
       		override fun onClick(v: View?) {
                
          }
        })
// 使用lambda，隐式创建实例，编译器帮我们转换成显式的代码
view.setOnClickListener{v->
    
}

```

当将这个接口作为一个方法参数时，需要分两步：

1. 创建一个实现类，并实现抽象方法（一般是匿名类）
2. 创建的实现类的实例

代码上，当形参类型是一个单一方法的接口时，可以用 lambda 表达式代替，但实际中，其形参还是需要一个具体的对象，所以 SAM-conversion就是，代码写了lambda，编译时帮我们转换成显式的代码，运行时创建接口的实例。



##  SAM对比类型别名（Type aliases）

与 Java 不同，Kotlin 支持函数类型，函数可以作为方法的形参。我们可以为一个函数类型定义一个别名，如下：

```kotlin
// 为一个函数类型取一个别名
typelias OnClick = (view) -> Unit

// 函数作为形参
fun View.setOnClick(onClick: OnClick)

```

虽然类型别名和SAM接口非常像，尤其在使用的时候，都可以写成lambda的形式，但其区别还是有的。

1. SAM效率效率低点，需要创建一个新的实例类，并创建实例。而类型别名不会创建新的类型
2. SAM也有优势，其接口类可以包含其他成员，比如其他非抽象成员，适用复杂的场景
   1. SAM适用和 Java无缝调用，而在 Kotlin 定义的类型别名，在Java中无妨使用
