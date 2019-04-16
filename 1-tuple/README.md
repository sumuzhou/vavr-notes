# 1-Tuple
Tuple是一种数据结构，可用于存放多个不同类型的对象。Vavr提供了从Tuple1到Tuple8的数据结构，并且是不可变的。

## 创建tuple
```java
// (Java, 8)
Tuple2<String, Integer> java8 = Tuple.of("Java", 8);
// "Java"
String s1 = java8._1;
// 8
Integer i2 = java8._2;
```
以Tuple2为例，通过Tuple.of工厂方法创建。Tuple2是泛型数据结构，需要指定类型，保证类型安全。获取时通过_1和_2直接读取，注意下标是从1开始的，而非从0开始。大家可能有疑问，不通过get方法直接读取属性不是违背了封装的原理吗？事实上并没有，Tuple2的_1和_2都是final的，直接读取是安全的；同时符合函数式编程的风格，在不带入参时可省略括号，刻意模糊函数和值之间的区别。

## 使用map
除了直接读取_1和_2，也可以通过map和apply方法操作Tuple2：
```java
// (vavr, 1)
Tuple2<String, Integer> t1 = java8.map(
        s -> s.substring(2) + "vr",
        i -> i / 8
);
// (vavr, "1")
Tuple2<String, String> t2 = java8.map(
        (s, i) -> Tuple.of(s.substring(2) + "vr", String.valueOf(i / 8))
);
// "vavr 1"
String applied = java8.apply(
        (s, i) -> s.substring(2) + "vr " + i / 8
);
```
map保持context不变，只转换里面的内容；即，返回的还是Tuple2，但泛型类型可以改变。这一点在后面的Option时会重点提及。apply则改变context，返回自定义的内容。map有两种方式，分别是传入两个Function和传入一个BiFunction。

## 源码阅读
Tuple2的源码相对简单，我们可以从头到尾过一遍。首先是类定义和属性域：
```java
public final class Tuple2<T1, T2> implements Tuple, Comparable<Tuple2<T1, T2>>, Serializable {
    public final T1 _1;

    public final T2 _2;

    public T1 _1() {
        return _1;
    }

    public T2 _2() {
        return _2;
    }
}
```
Tuple2接受T1、T2两个泛型，两者没有任何关系；并且实现了Tuple、比较和序列化三个接口。Tuple2的属性域都是final的。Tuple2同时提供了get方法，名称和属性域一样。函数式编程有个很重要的概念叫引用透明（后面还会讲），这里就是一个例子，使用值_1或者函数_1()，对程序而言没有任何区别；如前所述，在函数不带入参时，可省略括号。

Tuple2的方法基本可分为三类：用于修改数据的方法、用于转换数据的方法和用于比较的方法。

**修改数据**
```java
public Tuple2<T1, T2> update1(T1 value) {
    return new Tuple2<>(value, _2);
}

public Tuple2<T1, T2> update2(T2 value) {
    return new Tuple2<>(_1, value);
}

public Tuple2<T2, T1> swap() {
    return Tuple.of(_2, _1);
}
```
函数式编程很重要的特点，就是只使用常量而非变量，换句话说，不会修改已有数据。所以，上述修改数据的方法，都会返回新的Tuple2，而非修改已有Tuple2。后面讲集合的章节会单独讨论这种方式的利弊，这里不再展开。温馨提醒，写代码时一定记得使用返回值：
```java
// wrong
tup.swap();
// OK
tup = tup.swap();
```
**转换数据**
```java
public Map.Entry<T1, T2> toEntry() {
    return new AbstractMap.SimpleEntry<>(_1, _2);
}

@Override
public Seq<?> toSeq() {
    return List.of(_1, _2);
}

public <U1, U2> Tuple2<U1, U2> map(BiFunction<? super T1, ? super T2, Tuple2<U1, U2>> mapper) {
    Objects.requireNonNull(mapper, "mapper is null");
    return mapper.apply(_1, _2);
}

public <U1, U2> Tuple2<U1, U2> map(Function<? super T1, ? extends U1> f1, Function<? super T2, ? extends U2> f2) {
    Objects.requireNonNull(f1, "f1 is null");
    Objects.requireNonNull(f2, "f2 is null");
    return Tuple.of(f1.apply(_1), f2.apply(_2));
}

public <U> Tuple2<U, T2> map1(Function<? super T1, ? extends U> mapper) {
    Objects.requireNonNull(mapper, "mapper is null");
    final U u = mapper.apply(_1);
    return Tuple.of(u, _2);
}

public <U> Tuple2<T1, U> map2(Function<? super T2, ? extends U> mapper) {
    Objects.requireNonNull(mapper, "mapper is null");
    final U u = mapper.apply(_2);
    return Tuple.of(_1, u);
}

public <U> U apply(BiFunction<? super T1, ? super T2, ? extends U> f) {
    Objects.requireNonNull(f, "f is null");
    return f.apply(_1, _2);
}
```
toEntry用于和Map进行交互，在Tuple类中还有fromEntry方法，两者互补。toSeq将Tuple2中数据以Seq形式表现出来，关于Seq后面还会讲。然后就是map和apply方法，map方法有多个变种便于使用。

**比较**
```java
public static <T1, T2> Comparator<Tuple2<T1, T2>> comparator(Comparator<? super T1> t1Comp, Comparator<? super T2> t2Comp) {
    return (Comparator<Tuple2<T1, T2>> & Serializable) (t1, t2) -> {
        final int check1 = t1Comp.compare(t1._1, t2._1);
        if (check1 != 0) {
            return check1;
        }

        final int check2 = t2Comp.compare(t1._2, t2._2);
        if (check2 != 0) {
            return check2;
        }

        // all components are equal
        return 0;
    };
}

@SuppressWarnings("unchecked")
private static <U1 extends Comparable<? super U1>, U2 extends Comparable<? super U2>> int compareTo(Tuple2<?, ?> o1, Tuple2<?, ?> o2) {
    final Tuple2<U1, U2> t1 = (Tuple2<U1, U2>) o1;
    final Tuple2<U1, U2> t2 = (Tuple2<U1, U2>) o2;

    final int check1 = t1._1.compareTo(t2._1);
    if (check1 != 0) {
        return check1;
    }

    final int check2 = t1._2.compareTo(t2._2);
    if (check2 != 0) {
        return check2;
    }

    // all components are equal
    return 0;
}

@Override
public int compareTo(Tuple2<T1, T2> that) {
    return Tuple2.compareTo(this, that);
}
```
comparator方法相当于高阶函数，联合_1和_2两种类型各自的比较器，生成一个新的比较器。有两个地方值得注意，一是比较的顺序，优先比较_1，如果_1能比出大小就直接返回了；二是Comparator<Tuple2<T1, T2>> & Serializable这种写法，用来表示多接口，我也是头次见。

两个compareTo方法用于实现Comparable接口。其中private方法的泛型很有意思：<U1 extends Comparable<? super U1>, U2 extends Comparable<? super U2>>，为神马要这么写呢？

由于历史原因，Java的泛型使用了类型擦除（Type Erasure），在编译成bytecode后泛型就被Object替代。所以，不能同时实现两次仅仅是泛型不同的相同接口。说人话，如下代码是会报错的：
```java
static class Animal implements Comparable<Animal> {
  @Override
  public int compareTo(Animal o) {
    // TODO Auto-generated method stub
    return 0;
  }
}

static class Cat extends Animal implements Comparable<Cat> {

}
```
编译器会报错：The interface Comparable cannot be implemented more than once with different arguments。所以，我们只能选择在一个地方实现Comparable，一般会选择在父类，也就是Animal类中。为求简单，我们用Tuple1做个实验：
```java
@SuppressWarnings("unchecked")
  private static <U1 extends Comparable<? super U1>> int compareTo(Tuple1<?> o1, Tuple1<?> o2) {
      final Tuple1<U1> t1 = (Tuple1<U1>) o1;
      final Tuple1<U1> t2 = (Tuple1<U1>) o2;

      final int check1 = t1._1.compareTo(t2._1);
      if (check1 != 0) {
          return check1;
      }

      // all components are equal
      return 0;
  }

  private static <U1 extends Comparable<U1>> int compareTo2(Tuple1<U1> o1, Tuple1<U1> o2) {
      final int check1 = o1._1.compareTo(o2._1);
      if (check1 != 0) {
          return check1;
      }

      // all components are equal
      return 0;
  }
```
compareTo是源码的写法，compareTo2修改了泛型的声明形式。这两者的差别如下：
```java
// compareTo和compareTo2都OK
compareTo(animal, animal);
compareTo2(animal, animal);
// compareTo2编译不通过
compareTo(cat, cat);
compareTo(animal, cat);
compareTo2(cat, cat);
// compareTo运行出错
compareTo(animal, string);
```
compareTo2(cat, cat)编译不能通过，原因在于我们声明的是U1 extends Comparable<U1>，然而Cat并没有实现Comparable<Cat>接口；源码的Comparable<? super U1>就可以接受应对这种情况。但是使用wildcard也是有问题的，比如compareTo(animal, String)，编译时就不能检查出类型错误。所以，使用Comparator是不是更好呢？
