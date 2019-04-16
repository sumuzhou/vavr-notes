# vavr-notes
[![996.icu](https://img.shields.io/badge/link-996.icu-red.svg)](https://996.icu)
[![LICENSE](https://img.shields.io/badge/license-Anti%20996-blue.svg)](https://github.com/996icu/996.ICU/blob/master/LICENSE)
vavr流水账

最近发现一个很有趣的库Vavr，治好了多年对Scala的眼红病，跟大家分享一下。

众所周知，Java 8为我们带来了函数式编程，可以说革新了写代码的方式；其提供的Lambda、Stream、Optional以及CompletableFuture，个个都是编程利器。但是，和专业的函数式编程语言相比，总是感觉缺少了什么，比如：
- 多值返回：经常我们会遇到一个方法需要返回多个值的问题。传统的方法不外乎两种，一种是再包一层数据结构，然后作为单一对象返回；另一种是返回集合对象，比如List或Map。这两种方法都存在问题。第一种，偶尔一两次还行，多了就很浪费时间，增加代码量，并且平白增加了Classloader加载的类数量，占用Stack空间。第二种，多个值的类型很可能不同，在声明集合时，只能用Object作为泛型参数，这样在使用时就必须进行强制类型转换，还限制了编译器类型安全检查的能力，为类型错误留下了空间。
- 返回成功或异常：有首歌叫trouble is a friend，对程序员而言，异常Exception就是最常陪伴的朋友了。异常并不一定是坏事，特别是checked异常，有时还代表着业务逻辑。让我们困扰的，往往是unchecked异常改变了程序执行路线，导致出现不可预知的结果；从这个角度来说，异常和GOTO有点像，都导致了程序跳转执行。我们对程序的期望，是既能反映程序执行遇到的问题，又能返回合理的结果。这个时候我们就需要一个方法签名，来指明返回的是成功或异常。
- 一些其它问题：Stream的调用方式有些繁琐；Function只提供到BiFunction；case match还停留在石器时代；lambda表达式不能抛出checked异常；没有提供函数式的集合。

Vavr一次性满足了我们所有的愿望，比如提供了Tuple来封装多个值，提供了try和Either来处理异常。本文的部分内容摘自[Vavr的官方文档](http://www.vavr.io/vavr-docs)，想查阅第一手资料请移步官方文档。

Vavr的整体结构如下图所示：
![Vavr整体结构](https://github.com/sumuzhou/vavr-notes/blob/master/vavr-overview.png "Vavr整体结构")

后面我会按照Tuple、Function、Value和Traversable四个章节依次讲解Vavr的使用。
