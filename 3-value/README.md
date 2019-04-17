# 3-Value
Value是函数式编程的核心，粘贴vavr作者写在Value接口Javadoc中的一段话：
```
Functional programming is all about values and transformation of values using functions. The Value type reflects the values in a functional setting.
It can be seen as the result of a partial function application. Hence the result may be undefined. If a value is undefined, we say it is empty.
How the empty state is interpreted depends on the context, i.e. it may be undefined, failed, no elements, etc.
```
Vavr库的大部分类都建立在Value接口之上，Value接口继承了Iterable接口，这也是Vavr和Java核心库之间的关联，详细说明请移步[Javadoc](https://www.javadoc.io/doc/io.vavr/vavr/0.9.3)。

Vavr的Value都是不可变的，写操作会通过共享并创建新实例实现，这一点在后面的Traversable章节会详细描述。通过不可变，我们自动实现了线程安全。*但是注意，Value的一些方法是有副作用的；所谓的不可变，指的Value封装的值。*

## Option
Option类似Java的Optional，用于封装一个可能的值。和Optional不同的是，Option是一个接口，并显式的区分了Some和None两个实现类，通过代码可以看出区别：
```java
// Optional
Optional<String> maybeFooOpt = Optional.of("foo");
// false
maybeFooOpt.map(s -> (String) null).map(s -> s.toUpperCase() + "bar").isPresent();

// Option
Option<String> maybeFoo = Option.of("foo");
// Will throw NullPointerException
maybeFoo.map(s -> (String)null).map(s -> s.toUpperCase() + "bar").isEmpty();
```
为什么会出现这样的情况？Optional是一个二合一的类，将有值和无值的情况集成在一起，在内部逻辑会处理两种情况。而Option不同，区分了Some和None，使用map并不改变Context，也就是Some还是Some，而Some隐含的意义就是存在值。我们可以看下源码：
**Optional**
```java
public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        return Optional.ofNullable(mapper.apply(value));
    }
}
// ...
public boolean isPresent() {
    return value != null;
}
```
**Option**
```java
@Override
default <U> Option<U> map(Function<? super T, ? extends U> mapper) {
    Objects.requireNonNull(mapper, "mapper is null");
    return isEmpty() ? none() : some(mapper.apply(get()));
}
// ...
final class Some<T> implements Option<T>, Serializable {
  // ...
  @Override
  public boolean isEmpty() {
      return false;
  }
  // ...
}
```
Optional在map时，通过isPresent判断值是否为null；Option在map时，判断isEmpty，而Some的isEmpty返回false。对此作者是这么解释的：
```
This seems like Vavr’s implementation is broken, but in fact it’s not - rather, it adheres to the requirement of a monad to maintain computational context when calling .map.
In terms of an Option, this means that calling .map on a Some will result in a Some, and calling .map on a None will result in a None.
In the Java Optional example above, that context changed from a Some to a None.
This may seem to make Option useless, but it actually forces you to pay attention to possible occurrences of null and deal with them accordingly instead of unknowingly accepting them.
```
也就是说，Option更符合理论规范，并且会让编程人员注意可能出现的空值。*怎么说呢，就我而言，我还是更喜欢Optional，毕竟我用Optional就是因为不确定是否会有空值嘛，理论归理论，实际用着顺手就行。*

用Option如何处理map过程中可能出现的空值呢？解决方法是使用flatMap：
```java
Option<String> maybeFooBar = maybeFoo.map(s -> (String) null)
    .flatMap(s -> Option.of(s).map(t -> t.toUpperCase() + "bar"));
// or
Option<String> maybeFooBarToo = maybeFoo.flatMap(s -> Option.of((String) null))
    .map(s -> s.toUpperCase() + "bar");
```
flatMap允许Context的转换，可以从Some变成None。除了flatMap，Option还有一些有用的方法，让我们看下源码：
```java
@Override
default <U> Option<U> map(Function<? super T, ? extends U> mapper) {
    Objects.requireNonNull(mapper, "mapper is null");
    return isEmpty() ? none() : some(mapper.apply(get()));
}

@SuppressWarnings("unchecked")
default <U> Option<U> flatMap(Function<? super T, ? extends Option<? extends U>> mapper) {
    Objects.requireNonNull(mapper, "mapper is null");
    return isEmpty() ? none() : (Option<U>) mapper.apply(get());
}

default Option<T> filter(Predicate<? super T> predicate) {
    Objects.requireNonNull(predicate, "predicate is null");
    return isEmpty() || predicate.test(get()) ? this : none();
}

default <U> U transform(Function<? super Option<T>, ? extends U> f) {
    Objects.requireNonNull(f, "f is null");
    return f.apply(this);
}

@Override
default Option<T> peek(Consumer<? super T> action) {
    Objects.requireNonNull(action, "action is null");
    if (isDefined()) {
        action.accept(get());
    }
    return this;
}
```
从上面代码可以看出map和flatMap的区别，flatMap允许mapper自定义返回的Option是Some还是None，而map直接将mapper的返回包装在Some中。其它方法，filter方法对Option的值做判断，如果是Some且满足条件，返回该Some，否则返回None；transform通过函数将Option转化为一个值；peek消费Option内的值，但不改变Option。

## Try
Try是一种封装类型，用来表示一段逻辑的执行结果，是成功执行返回结果，还是发生了错误。和Option类似，Try也是一个接口，有两个实现类：Success和Failure，分别表示成功或失败。Try一般通过工厂方法of创建：
```java
// Try
// Success(3.0)
Try.of(() -> Math.sqrt(9));
// Failure(java.nio.file.NoSuchFileException: somefile)
Try.of(() -> Files.newInputStream(Paths.get("somefile")).available());
```
工厂方法通过执行ChechedFunction0，得到结果并保存在Try中。看下源码：
```java
static <T> Try<T> of(CheckedFunction0<? extends T> supplier) {
    Objects.requireNonNull(supplier, "supplier is null");
    try {
        return new Success<>(supplier.apply());
    } catch (Throwable t) {
        return new Failure<>(t);
    }
}
```
上面的newInputStream示例存在安全隐患，因为我们使用完成后或出错后需要关闭输入流。对此有另外一种实现方式：
```java
Try.withResources(() -> Files.newInputStream(Paths.get("somefile"))).of(is -> is.available());
```
将需要关闭的资源包装成supplier传递给withResources，然后通过of方法声明如何使用资源。代码实现很简单，就是将supplier提取的资源放在try语句中：
```java
@SuppressWarnings("try")/* https://bugs.openjdk.java.net/browse/JDK-8155591 */
public <R> Try<R> of(CheckedFunction1<? super T1, ? extends R> f) {
    return Try.of(() -> {
        try (T1 t1 = t1Supplier.apply()) {
            return f.apply(t1);
        }
    });
```
如何从Try中拿到计算值呢？最直接的方法就是get，但是要注意，如果是Failure，则会抛出异常。所以更安全的方法是使用getOrElse，或者将Try转成Either。

Try更强大的功能是recover，可以根据指定的错误类型得到不同的结果，结合Vavr提供的Case Match能力，十分强大：
```java
String result = Try.of((CheckedFunction0<String>)() -> { throw new IllegalArgumentException("参数错了"); })
    .recover(x -> Match(x).of(
        Case($(instanceOf(IllegalArgumentException.class)), t -> "发生错误：" + t.getMessage()),
        Case($(instanceOf(NoSuchElementException.class)), t -> "发生错误：" + t.getMessage())))
    .getOrElse("Can not recover");
```
Case Match后面会提到，我们只需要注意recover其实接受一个入参为Throwable的Function，在出错时执行。看下源码：
```java
default Try<T> recover(Function<? super Throwable, ? extends T> f) {
    Objects.requireNonNull(f, "f is null");
    if (isFailure()) {
        return Try.of(() -> f.apply(getCause()));
    } else {
        return this;
    }
}
```

## Either
