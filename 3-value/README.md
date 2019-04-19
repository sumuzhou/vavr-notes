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
import static io.vavr.API.*;        // $, Case, Match
import static io.vavr.Predicates.*; // instanceOf

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
Either表示可能存在两种情况，但最终是其中一种情况。Either是一个接口，有两个实现类Left和Right。一般而言，Either用于表示可能返回的结果或者可能出现的错误，和Try有些类似；约定俗成的，Left被认为是出现了错误，Right被认为是正确的结果。所以Try提供了toEither方法：
```java
default Either<Throwable, T> toEither() {
    if (isFailure()) {
        return Either.left(getCause());
    } else {
        return Either.right(get());
    }
}
```
从上述代码可看到，Either没有提供of方法，取而代之是left和right方法。

也因为约定俗成，Either的map方法只对Right生效。什么意思呢？比如如下代码：
```java
// Left("Error")
Either<String,Integer> left = Either.<String, Integer>left("Error").map(i -> i * 2);
// Right(2)
Either<String,Integer> right = Either.<String, Integer>right(1).map(i -> i * 2);
```
left经过map后并没有发生变化，反观right是起了作用。看下源码：
```java
@SuppressWarnings("unchecked")
@Override
default <U> Either<L, U> map(Function<? super R, ? extends U> mapper) {
    Objects.requireNonNull(mapper, "mapper is null");
    if (isRight()) {
        return Either.right(mapper.apply(get()));
    } else {
        return (Either<L, U>) this;
    }
}
```
很明显，显式判断了isRight，否则是不变的。那如果要处理Left该怎么办呢？使用mapLeft方法即可：
```java
@SuppressWarnings("unchecked")
default <U> Either<U, R> mapLeft(Function<? super L, ? extends U> leftMapper) {
    Objects.requireNonNull(leftMapper, "leftMapper is null");
    if (isLeft()) {
        return Either.left(leftMapper.apply(getLeft()));
    } else {
        return (Either<U, R>) this;
    }
}
```

## Lazy
顾名思义，Lazy就是懒加载的对象，封装了计算逻辑，第一次使用时才进行计算，后续使用保存的计算结果。常见Lazy使用of工厂方法，提供一个Supplier：
```java
Lazy<Double> lazy = Lazy.of(Math::random);
lazy.isEvaluated(); // = false
lazy.get();         // = 0.123 (random generated)
lazy.isEvaluated(); // = true
lazy.get();         // = 0.123 (memoized)
```
Lazy和Supplier很类似（其实Lazy继承了Supplier接口），区别是Lazy会保存计算结果。但是保存就有些小技巧在里面了，请看源码：
```java
private transient volatile Supplier<? extends T> supplier;
private T value;

@Override
public T get() {
    return (supplier == null) ? value : computeValue();
}

private synchronized T computeValue() {
    final Supplier<? extends T> s = supplier;
    if (s != null) {
        value = s.get();
        supplier = null;
    }
    return value;
}
```
这是一个很经典的Double-Checked Locking问题，关键点在于supplier的volatile关键字，在supplier的读写之间建立了happens-before的关系，避免重排带来的问题。头一次的supplier = null写操作和后一次的final Supplier<? extends T> s = supplier读操作之间建立了happens-before的关系，后续读取supplier的时候，JVM的内存模型确保了头一次的写入内容是可见的；也就是说，如果supplier是null，那么value的值一定是经过计算的。这也就是value不需要标记为volatile的原因。更详细的解释可参考作者在[源码中给出的链接](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#volatile)。

回到Lazy本身，我们创建的Lazy是封装类型，那能否创建实际类型的懒加载值呢？答案是可以的，通过代理来创建：
```java
CharSequence chars = Lazy.val(() -> "Yay!", CharSequence.class);
```
看下源码的实现原理：
```java
@GwtIncompatible("reflection is not supported")
@SuppressWarnings("unchecked")
public static <T> T val(Supplier<? extends T> supplier, Class<T> type) {
    Objects.requireNonNull(supplier, "supplier is null");
    Objects.requireNonNull(type, "type is null");
    if (!type.isInterface()) {
        throw new IllegalArgumentException("type has to be an interface");
    }
    final Lazy<T> lazy = Lazy.of(supplier);
    final InvocationHandler handler = (proxy, method, args) -> method.invoke(lazy.get(), args);
    return (T) Proxy.newProxyInstance(type.getClassLoader(), new Class<?>[] { type }, handler);
}
```
原理并不复杂，还是先创建Lazy对象，然后创建代理，代理做的事情就是将方法调用传递给lazy.get，即实际的懒加载对象。

## Validation
顾名思义Validation就是用来做验证的。Validation和Either很类似，有两个实现类Valid和Invalid，基本等同于Either的Right和Left；Validation提供的map和mapError方法，等同于Either的map和mapLeft方法。区别在于，Validation的目的是用于验证，当验证时，我们肯定希望一次性验证多项属性，而不是当某属性出错时就返回结果。为此，Validation提供了combine和ap方法，来串联单项属性的验证结果，如下代码：
```java
class Person {

    public final String name;
    public final int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person(" + name + ", " + age + ")";
    }

}

class PersonValidator {

    private static final String VALID_NAME_CHARS = "[a-zA-Z ]";
    private static final int MIN_AGE = 0;

    public Validation<Seq<String>, Person> validatePerson(String name, int age) {
        return Validation.combine(validateName(name), validateAge(age)).ap(Person::new);
    }

    private Validation<String, String> validateName(String name) {
        return CharSeq.of(name).replaceAll(VALID_NAME_CHARS, "").transform(seq -> seq.isEmpty()
                ? Validation.valid(name)
                : Validation.invalid("Name contains invalid characters: '"
                + seq.distinct().sorted() + "'"));
    }

    private Validation<String, Integer> validateAge(int age) {
        return age < MIN_AGE
                ? Validation.invalid("Age must be at least " + MIN_AGE)
                : Validation.valid(age);
    }

}

// Validation
PersonValidator personValidator = new PersonValidator();
// Valid(Person(John Doe, 30))
Validation<Seq<String>, Person> valid = personValidator.validatePerson("John Doe", 30);
// Invalid(List(Name contains invalid characters: '!4?', Age must be greater than 0))
Validation<Seq<String>, Person> invalid = personValidator.validatePerson("John? Doe!4", -1);
// Invalid(List(Age must be at least 0))
Validation<Seq<String>, Person> alsoInvalid = personValidator.validatePerson("John Doe", -1);
```
可以看到，我们通过validateName和validateAge实现了单项属性的验证，然后通过combine和ap把验证结果合并起来；当所有属性验证通过时返回创建对象，当存在属性验证失败时返回所有验证失败的内容。看下combine和ap的源码是如何实现的：
```java
static <E, T1, T2> Builder<E, T1, T2> combine(Validation<E, T1> validation1, Validation<E, T2> validation2) {
    Objects.requireNonNull(validation1, "validation1 is null");
    Objects.requireNonNull(validation2, "validation2 is null");
    return new Builder<>(validation1, validation2);
}

final class Builder<E, T1, T2> {
    // ...
    private Builder(Validation<E, T1> v1, Validation<E, T2> v2) {
        this.v1 = v1;
        this.v2 = v2;
    }

    public <R> Validation<Seq<E>, R> ap(Function2<T1, T2, R> f) {
        return v2.ap(v1.ap(Validation.valid(f.curried())));
    }
    // ...
}

default <U> Validation<Seq<E>, U> ap(Validation<Seq<E>, ? extends Function<? super T, ? extends U>> validation) {
    Objects.requireNonNull(validation, "validation is null");
    if (isValid()) {
        if (validation.isValid()) {
            final Function<? super T, ? extends U> f = validation.get();
            final U u = f.apply(this.get());
            return valid(u);
        } else {
            final Seq<E> errors = validation.getError();
            return invalid(errors);
        }
    } else {
        if (validation.isValid()) {
            final E error = this.getError();
            return invalid(List.of(error));
        } else {
            final Seq<E> errors = validation.getError();
            final E error = this.getError();
            return invalid(errors.append(error));
        }
    }
}
```
combine并不神奇，只是创建一个Builder；Builder的ap方法，将传入函数柯里化，并包装在Valid中，然后倒序（也就是v2 -> v1）调用被combine的Validation的ap方法。回忆一下，柯里化是将多参数的函数转变为多个单参数函数的串联。再看下Validation的ap方法，对两者的状态做判断，只要有一个是Invalid，最终都是Invalid；多个Invalid通过List的append方法合并起来。

## Match
Scala提供了强大的类型匹配能力，非常好用，比如：
```scala
val s = i match {
  case 1 => "one"
  case 2 => "two"
  case _ => "?"
}
```
除了基础的匹配，Scala还有以下能力：
- 命名参数：case i: Int ⇒ "Int " + i
- 对象分解：case Some(i) ⇒ i
- 条件判断：case Some(i) if i > 0 ⇒ "positive " + i
- 多条件判断：case "-h" | "--help" ⇒ displayHelp
- 编译时可检查穷举性（*没有尝试过，请自行判断正误*）

Vavr提供了类似的类型匹配能力，基础的匹配用Vavr实现如下：
```java
import static io.vavr.API.*;
import static io.vavr.Predicates.*;
import static io.vavr.Patterns.*;

String s = Match(i).of(
    Case($(1), "one"),
    Case($(2), "two"),
    Case($(), "?")
);
```
注意上述的静态引用，使用Vavr的类型匹配时一般都会用到。看下源码是如何实现的：
```java
// API.Match
public static <T> Match<T> Match(T value) {
    return new Match<>(value);
}
```
静态方法Match很简单，就是创建一个Match对象。注意方法名是大写开头，不符合驼峰原则，这是因为case是Java的关键字，所以后续的静态方法Case就用了大写开头；为保持风格一致，这里的Match也用大写开头。
```java
public static final class Match<T> {

    private final T value;

    private Match(T value) {
        this.value = value;
    }

    public final <R> R of(Case<? extends T, ? extends R>... cases) {
        Objects.requireNonNull(cases, "cases is null");
        for (Case<? extends T, ? extends R> _case : cases) {
            final Case<T, R> __case = (Case<T, R>) _case;
            if (__case.isDefinedAt(value)) {
                return __case.apply(value);
            }
        }
        throw new MatchError(value);
    }
    // ...
}
```
of方法接受多个Case对象，对每个Case对象，判断是否接受值；如果能，则用Case的apply方法处理；如果到最后还不能处理，则抛出异常。Case类的方法看着是不是非常眼熟？对的，Case类就是一个Partial Function：
```java
public interface Case<T, R> extends PartialFunction<T, R> {
}
```
那么，Case($(1), "one")这段代码做了什么呢？
```java
public static <T, R> Case<T, R> Case(Pattern0<T> pattern, R retVal) {
    Objects.requireNonNull(pattern, "pattern is null");
    return new Case0<>(pattern, ignored -> retVal);
}

public static <T> Pattern0<T> $(T prototype) {
    return new Pattern0<T>() {
        @Override
        public T apply(T obj) {
            return obj;
        }

        @Override
        public boolean isDefinedAt(T obj) {
            return Objects.equals(obj, prototype);
        }
    };
}
```
Case方法接受一个Pattern0和返回值，并创建一个Case0类。$方法（对没错，$是一个方法的名称）返回一个Pattern0。再来看下Pattern0和Case0：
```java
public static final class Case0<T, R> implements Case<T, R> {

    private final Pattern0<T> pattern;
    private final Function<? super T, ? extends R> f;

    private Case0(Pattern0<T> pattern, Function<? super T, ? extends R> f) {
        this.pattern = pattern;
        this.f = f;
    }

    @Override
    public R apply(T obj) {
        return f.apply(pattern.apply(obj));
    }

    @Override
    public boolean isDefinedAt(T obj) {
        return pattern.isDefinedAt(obj);
    }
}

public interface Pattern<T, R> extends PartialFunction<T, R> {
}

public static abstract class Pattern0<T> implements Pattern<T, T> {
  // 。。。
}
```
Case0和Pattern0都是Partial Function，所以两者都有isDefinedAt和apply方法，Case0在Pattern0之上封装了一层。回到最初的Match - Case语句，假设i为1，of循环运行到第一个Case时，判断$(1)创建的Pattern，isDefinedAt通过，调用apply方法，也就是返回1本身；然后调用Case的apply方法，也就是ignored -> retVal，所以最后返回"one"。

除了基础匹配，Vavr也提供了类似Scala的匹配能力。

**命名参数**
```java
Number plusOne = Match(obj).of(
    Case($(instanceOf(Integer.class)), i -> i + 1),
    Case($(instanceOf(Double.class)), d -> d + 1),
    Case($(), o -> { throw new NumberFormatException(); })
);
```
**对象分解**
```java
Match(option).of(
      Case($Some($(1)), i -> "Int " + i),
      Case($Some($()), "defined"),
      Case($None(), "empty")
);
```
很神奇，来看下源码怎么实现的：
```java
public static <T, _1 extends T> Pattern1<Option.Some<T>, _1> $Some(Pattern<_1, ?> p1) {
    return Pattern1.of(Option.Some.class, p1, io.vavr.$::Some);
}
```
$Some方法通过of方法创建了一个Pattern1，看下Pattern1的代码：
```java
public static abstract class Pattern1<T, T1> implements Pattern<T, T1> {

    public static <T, T1 extends U1, U1> Pattern1<T, T1> of(Class<? super T> type, Pattern<T1, ?> p1, Function<T, Tuple1<U1>> unapply) {
        return new Pattern1<T, T1>() {
            @SuppressWarnings("unchecked")
            @Override
            public T1 apply(T obj) {
                return (T1) unapply.apply(obj)._1;
            }

            @SuppressWarnings("unchecked")
            @Override
            public boolean isDefinedAt(T obj) {
                if (obj == null || !type.isAssignableFrom(obj.getClass())) {
                    return false;
                } else {
                    final Tuple1<U1> u = unapply.apply(obj);
                    return
                            ((Pattern<U1, ?>) p1).isDefinedAt(u._1);
                }
            }
        };
    }
}
```
关键点在于unapply，用途是从数据结构中提取出值，后续流程就基本类似了。所以$Some的unapply方法用的是什么呢？是io.vavr.$::Some，对别看错了，$是一个类的名字。看下这段代码：
```java
@Unapply
static <T> Tuple1<T> Some(Option.Some<T> some) { return Tuple.of(some.get()); }
```
很简单吧，就是从Some中拿出数据封装成Tuple1。注意@Patterns和@Unapply注解，根据作者的描述，我们也是可以自定义分解方法的，只需要在类定义加上@Patterns注解，在方法加上@Unapply注解，然后会通过annotation processor来生成对应代码。未进行尝试，有兴趣的同学可自行尝试。

对象分解的匹配表达式可以指定多层，但分解出来的对象限定在第一层，什么意思，请看如下代码：
```java
Match(_try).of(
    Case($Success($Tuple2($("a"), $())), tuple2 -> ...),
    Case($Failure($(instanceOf(Error.class))), error -> ...)
);
```
上述代码的Success虽然表达式内部的Tuple2，但分解出来的对象还是tuple2，而不是内部的"a"或者其它。

**条件判断**
```java
Match(option).of(
      Case($Some($(is(1))), j -> "positive " + j),
      Case($(), "?")
  );
```
Vavr只提供了is判断条件，想要模拟Scala的case Some(i) if i > 0条件判断，需要自定义判断条件：
```java
public static <T> Predicate<T> gt(Comparable<? super T> value) {
    return obj -> value.compareTo(obj) < 0;
}

Match(i).of(
      Case($Some($(gt(1))), j -> "positive " + j),
      Case($(), "?")
  );
```
**多条件判断**
```java
Match(arg).of(
      Case($(isIn("-h", "--help")), displayHelp),
      Case($(), "Invalid arguments")
  );
```
**避免MatchError**
因为Java并没有对类型匹配提供原生的支持，所以不能在编译时检查穷举性，不注意的话可能由于Case没有全覆盖导致抛出MatchError。解决办法是使用option方法代替of方法：
```java
Option<String> s = Match(i).option(
    Case($(0), "zero")
);
```
这样在没有匹配到时，返回结果是None。

## Future && Promise
Vavr也提供了Future和Promise类，用于异步编程。没有深入研究，还未发现其比CompletableFuture的优势。
