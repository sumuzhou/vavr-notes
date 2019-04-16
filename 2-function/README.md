# 2-Function
函数式编程的核心就是函数Function。Java 8提供了Function和BiFunction接口，分别对应一个入参和两个入参。Vavr在此基础上，扩展到了八个入参，从Function0到Function8。如果在函数内部想抛出受检异常，则可以使用CheckedFunction0到CheckedFunction8。

创建Function有三种方式，如下：
```java
// f1.apply(1, 2) = 3
Function2<Integer, Integer, Integer> f1 = (a, b) -> a + b;
Function2<Integer, Integer, Integer> f2 = Function2.of((a, b) -> a + b);
Function2<Integer, Integer, Integer> f3 = new Function2<Integer, Integer, Integer>() {
    @Override
    public Integer apply(Integer a, Integer b) {
        return a + b;
    }
};
```
可以直接从lambda表达式创建；也可以通过工厂方法of创建，of方法的入参可以是其它函数的引用（Method Reference）；还可以通过匿名类创建。

Vavr的函数提供了四种特性：
- Composition
- Lifting
- Currying
- Memoization

## Composition
函数可以合成，比如两个函数f: X → Y和g: Y → Z，可以合成h: g(f(x)) ，也就是h: X → Z。Vavr提供的合成手段有两种：
```java
// Composition
Function1<Integer, Integer> plusOne = a -> a + 1;
Function1<Integer, Integer> multiplyByTwo = a -> a * 2;
Function1<Integer, Integer> add1AndThenMultiplyBy2 = plusOne.andThen(multiplyByTwo);
Function1<Integer, Integer> multiplyBy2ComposeAdd1 = multiplyByTwo.compose(plusOne);
```
请注意andThen和compose的顺序，两者相反。另外只有Function1提供了compose方法。为什么呢，原因很简单，compose是先执行入参函数，然后再执行本身；入参函数产生的结果只有一个值，只能传递给Function1。看一下方法的源码：
```java
default <V> Function1<T1, V> andThen(Function<? super R, ? extends V> after) {
    Objects.requireNonNull(after, "after is null");
    return (t1) -> after.apply(apply(t1));
}

default <V> Function1<V, R> compose(Function<? super V, ? extends T1> before) {
    Objects.requireNonNull(before, "before is null");
    return v -> apply(before.apply(v));
}
```
很简单，就是做了一个嵌套调用，作为新的函数返回。

# Lifting
讲Lifting之前首先要了解什么是partial function（偏函数？）。在数学上，我们所谓的函数，是指将一个集合X映射到另一个集合Y的确定方式，适用于X中的每一个元素。偏函数就是指f: X′ → Y，其中X′是X的子集，意味着偏函数只能映射X中的部分元素。如果入参传入了不支持的元素，偏函数会报错。
```java
Function2<Integer, Integer, Integer> divide = (a, b) -> a / b;
```
上面的divide就是一个偏函数，因为只支持非0的元素。

Lifting的作用，就是将偏函数转化为完备函数：
```java
Function2<Integer, Integer, Option<Integer>> safeDivide = Function2.lift(divide);
// = None
Option<Integer> i1 = safeDivide.apply(1, 0);
// = Some(2)
Option<Integer> i2 = safeDivide.apply(4, 2);
```
我们看下源码：
```java
@SuppressWarnings("RedundantTypeArguments")
static <T1, T2, R> Function2<T1, T2, Option<R>> lift(BiFunction<? super T1, ? super T2, ? extends R> partialFunction) {
    return (t1, t2) -> Try.<R>of(() -> partialFunction.apply(t1, t2)).toOption();
}
```
没有什么黑科技，原理很简单，把偏函数包装在Try里面，然后返回对象使用Option；如果支持，返回Some，否则返回None。关于Try和Option，后面讲Value的时候会讲到。

## Currying
Currying（柯里化？）和partial application（中文要怎么翻译？）是两个类似的概念。Partial application指通过固定部分参数，从函数中派生出的新函数。
```java
// Partial application
Function2<Integer, Integer, Integer> sum2 = (a, b) -> a + b;
Function1<Integer, Integer> add2 = sum2.apply(2);
```
上述代码中，sum2是原本的函数，通过固定第一个整数为2，派生出了add2这个partial application。

而所谓Currying，是指将多参数函数拆分成多个单参数函数的合成：
```java
// Currying
Function2<Integer, Integer, Integer> sum2 = (a, b) -> a + b;
Function1<Integer, Integer> add2Curried = sum2.curried().apply(2);
```
上述代码将sum2函数柯里化，然后固定了第一个整数为2，生成新的Function1。这个示例看着和partial application一样，参数多了就能看出区别：
```java
Function3<Integer, Integer, Integer, Integer> sum3 = (a, b, c) -> a + b + c;
// Partial application
Function2<Integer, Integer, Integer> add3 = sum3.apply(3);
// Currying
Function1<Integer, Function1<Integer, Integer>> add3Curried = sum3.curried().apply(3);
```
注意add3和add3Curried的类型，partial application派生出新的函数，而柯里化拆分成单参数函数的合成。看下Function3中的源码：
```java
default Function2<T2, T3, R> apply(T1 t1) {
    return (T2 t2, T3 t3) -> apply(t1, t2, t3);
}

default Function1<T3, R> apply(T1 t1, T2 t2) {
    return (T3 t3) -> apply(t1, t2, t3);
}

@Override
default Function1<T1, Function1<T2, Function1<T3, R>>> curried() {
    return t1 -> t2 -> t3 -> apply(t1, t2, t3);
}
```
Function3可以固定一个或两个参数派生partial application；另外注意curried方法的返回语句，说明了lambda表达式是从右往左估值的。

## Memoization
Memoization（记忆化？）是一种缓存手段，简而言之，实现了记忆化的函数在首次调用时会进行计算，后续调用直接从缓存中获取。记忆化常用于计算密集型方法，尤其是DP、递归等方法。Vavr提供了记忆化的包装方法：
```java
// Memoization
Function0<Double> hashCache = Function0.of(Math::random).memoized();
double randomValue1 = hashCache.apply();
double randomValue2 = hashCache.apply();
```
上述的randomValue1和randomValue2值是一致的，因为第二次调用没有进行计算。看下源码，以Function2为例：
```java
@Override
default Function2<T1, T2, R> memoized() {
    if (isMemoized()) {
        return this;
    } else {
        final Map<Tuple2<T1, T2>, R> cache = new HashMap<>();
        return (Function2<T1, T2, R> & Memoized) (t1, t2)
                -> Memoized.of(cache, Tuple.of(t1, t2), tupled());
    }
}

interface Memoized {
    static <T extends Tuple, R> R of(Map<T, R> cache, T key, Function1<T, R> tupled) {
        synchronized (cache) {
            if (cache.containsKey(key)) {
                return cache.get(key);
            } else {
                final R value = tupled.apply(key);
                cache.put(key, value);
                return value;
            }
        }
    }
}
```
上述是两处不同的代码，放在一起方便阅读。memoized方法将原始函数通过of工厂方法包装一层，并通过类型转换增加了Memoized接口实现。如果后续再调用memoized，就会直接return this。我们看of方法的入参，其实缓存就是一个Map，缓存的key就是原始函数入参组成的Tuple2，所以，对相同的入参，就不必再计算，直接从缓存中获取。有个细节，这里使用了tupled方法将Function2改成了Function1，方便统一使用Tuple2。

使用memoized有两点需要注意：1. 原始方法的入参需要定义好hashCode，才能确保找到缓存；2. 记忆化实际上产生了side-effect，改变了状态，所以为确保线程安全使用了synchronized加锁；实际使用中记忆化应该读远多于写，是否可以使用读写分离锁或乐观锁来提高性能？
