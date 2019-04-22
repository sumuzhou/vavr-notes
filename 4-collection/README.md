# Collection
Vavr基于Lambda设计了一套全新的Collection库，实现了函数式数据结构，包含几个重要的特征：不可变、持久化和引用透明。Vavr和Java的Collection共同点仅在于都继承了Iterable接口。接下来讲解Vavr集合的概念。

## 可变数据结构
由于历史原因，Java集合设计为可变的数据结构，包含了多个void方法，比如：
```java
interface Collection<E> {
    // removes all elements from this collection
    void clear();
}
```
void方法是很独特的方法，表明一定会产生副作用（否则调用方法的意义何在），并且可能伴随着状态修改。修改共享状态是程序错误的一大重要来源，不仅仅是在并发情况下。

## 不可变数据结构
顾名思义，不可变数据结构就是创建后不可改变的数据结构。不可变带来的最直接影响就是线程安全，可以任意在线程中传递使用。Java中我们一般通过给集合包装不可变的视图来实现：
```java
List<String> list = Collections.unmodifiableList(otherList);

// Boom!
list.add("why not?");
```
注意包装后只能使用list，otherList仍然是可变的，如果其它地方存有otherList的引用，并不能保证不可变性。包装后，调用修改方法会抛出运行时异常。

## 持久化数据结构
持久化数据结构在修改时会保留之前版本的信息，所以是等效不可变的。完全的持久化结构允许在任意版本上修改和查询。试想一下，如果一个数据结构要允许对历史版本操作，那么它一定会保留之前的结构，即，数据只会创建，不会修改。后面介绍Vavr集合时有更清晰展示。

很多操作其实都只产生小变化，把之前版本全部拷贝一遍太低效了，所以，重点是要在两个版本之间找到相似性，并尽量共享数据。

## 引用透明
通俗来讲，一个函数调用，如果替换成函数调用的值，并不影响程序的行为，就可以说这个函数是引用透明的，比如：
```java
// not referentially transparent
Math.random();

// referentially transparent
Math.max(1, 2);
```
引用透明使推理和单元测试都变简单，在某些地方可以直接用值代替函数调用，减少对整体的依赖。

## Linked list
下面以单链表为例说明函数式数据结构。传统的单链表将链表视为多个链接节点的组合，而函数式单链表将链表视为一个头元素和尾部链表的组合。比如如下代码：
```java
// = List(1, 2, 3)
List<Integer> list1 = List.of(1, 2, 3);
// = List(0, 2, 3)
List<Integer> list2 = list1.tail().prepend(0);
```
形成的结果如图所示：
![list1结构](https://github.com/sumuzhou/vavr-notes/blob/master/vavr-overview.png "list1结构")

![list2结构](https://github.com/sumuzhou/vavr-notes/blob/master/vavr-overview.png "list2结构")

链表最后端的元素是Nil，一个特殊的类。可以看到，虽然list2调用了list1的tail和prepend方法，但是list1的数据并没有变化。list1和list2形成了一个更大的数据结构，而它们各自都是这个数据结构的一个局部视图（头元素决定入口）。

tail和prepend操作只需常数级时间，但其它大部分操作都需要线性时间。所以，List标记为LinearSeq的子接口，表明时间特性。

函数式单链表的原理很简单，我们可以看下源码：
```java
static <T> List<T> of(T... elements) {
    Objects.requireNonNull(elements, "elements is null");
    List<T> result = Nil.instance();
    for (int i = elements.length - 1; i >= 0; i--) {
        result = result.prepend(elements[i]);
    }
    return result;
}
```
of方法拿了一个Nil的实例，然后通过prepend逆向将元素加入到链表中。
```java
final class Nil<T> implements List<T>, Serializable {
  public static <T> Nil<T> instance() {
    return (Nil<T>) INSTANCE;
  }
}
```
Nil类是一个空链表的实现，同时提供了Nil单例。
```java
default List<T> prepend(T element) {
    return new Cons<>(element, this);
}
```
prepend方法就更简单了，就是创建一个新的Cons实例。
```java
final class Cons<T> implements List<T>, Serializable {
  private final T head;
  private final List<T> tail;
  private final int length;

  private Cons(T head, List<T> tail) {
    this.head = head;
    this.tail = tail;
    this.length = 1 + tail.length();
  }
}
```
Cons类是链表的实现，构造器就是设置头元素、尾部链表以及记录长度。所以对函数式单链表来说，链表就是头元素和尾部链表的组合。

## Queue
函数式队列可基于两个链表实现。第一个链表存储待出队的元素，第二个链表存储新入队的元素。当第一个链表为空时，将第二个链表反向作为第一个链表，比如：
```java
Queue<Integer> queue = Queue.of(1, 2, 3)
                            .enqueue(4)
                            .enqueue(5);
```
得到的数据结构如图所示：
![queue1结构](https://github.com/sumuzhou/vavr-notes/blob/master/vavr-overview.png "queue1结构")

出队前三个元素后，数据结构如图所示：
![queue2结构](https://github.com/sumuzhou/vavr-notes/blob/master/vavr-overview.png "queue2结构")

队列一般用法如下：
```java
Queue<Integer> queue = Queue.of(1, 2, 3);
// = (1, Queue(2, 3))
Tuple2<Integer, Queue<Integer>> dequeued =
        queue.dequeue();

// = Some((1, Queue()))
Queue.of(1).dequeueOption();        
// = None
Queue.empty().dequeueOption();

// = Some((1, Queue(2, 3)))
Option<Tuple2<Integer, Queue<Integer>>> dequeuedOpt =
        queue.dequeueOption();
// = Some(1)
Option<Integer> element = dequeuedOpt.map(Tuple2::_1);
// = Some(Queue(2, 3))
Option<Queue<Integer>> remaining =
    dequeuedOpt.map(Tuple2::_2);
```
因为是函数式队列，出队入队操作都是返回新的队列，而不改变原有队列，所以出队时返回值是Tuple2，包含出队值和新的队列。另外一个安全的方式是使用Option，就不会因为队列为空导致抛出异常。

队列的源代码非常简单，看一下：
```java
// 构造器
private Queue(io.vavr.collection.List<T> front, io.vavr.collection.List<T> rear) {
    final boolean frontIsEmpty = front.isEmpty();
    this.front = frontIsEmpty ? rear.reverse() : front;
    this.rear = frontIsEmpty ? front : rear;
}

// 入队
public Queue<T> enqueue(T element) {
    return new Queue<>(front, rear.prepend(element));
}

// 出队
public Tuple2<T, Q> dequeue() {
    if (isEmpty()) {
        throw new NoSuchElementException("dequeue of empty " + getClass().getSimpleName());
    } else {
        return Tuple.of(head(), tail());
    }
}

// head方法
public T head() {
    if (isEmpty()) {
        throw new NoSuchElementException("head of empty " + stringPrefix());
    } else {
        return front.head();
    }
}

// tail方法
public Queue<T> tail() {
    if (isEmpty()) {
        throw new UnsupportedOperationException("tail of empty " + stringPrefix());
    } else {
        return new Queue<>(front.tail(), rear);
    }
}
```
队列的构造器就决定了如果front为空，则将rear逆序作为front；入队只是在rear链表头增加元素；出队返回Tuple2，其中的值来自front的头元素，新队列来自front的尾部链表和rear链表。这段简洁的代码非常有美感。

## Sorted Set
函数式有序集基于二叉树结构创建，每个节点包含最多两个子节点和一个值；左子树所有值都小于节点值，右子树所有值大于节点值；查询的时间是log级别的。建立的二叉树如图所示：
```java
// = TreeSet(1, 2, 3, 4, 6, 7, 8)
SortedSet<Integer> xs = TreeSet.of(6, 1, 3, 2, 4, 7, 8);
```
![tree1结构](https://github.com/sumuzhou/vavr-notes/blob/master/vavr-overview.png "tree1结构")

增加元素是一个有趣的操作，因为是函数式有序集，需要保留原有内部结构，所以增加元素会添加一条从新增节点到根节点的路径，其它保持不变：
```java
// = TreeSet(1, 2, 3, 4, 5, 6, 7, 8)
SortedSet<Integer> ys = xs.add(5);
```
![tree2结构](https://github.com/sumuzhou/vavr-notes/blob/master/vavr-overview.png "tree2结构")

删除元素比增加更为复杂，另外树的左右深度需要保持基本一致水平。详细的理论请移步[Red/Black Tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree)和[Purely Functional Data Structures](https://www.cs.cmu.edu/~rwh/theses/okasaki.pdf)。

## Overview
Vavr提供了Seq、Set和Map接口以及多种实现，通过多种fluent方法减少了代码量。Seq接口及实现如图：
![seq结构](https://github.com/sumuzhou/vavr-notes/blob/master/vavr-overview.png "seq结构")

比如组装字符串的方法，之前的方法可能是：
```java
String join(String... words) {
    StringBuilder builder = new StringBuilder();
    for(String s : words) {
        if (builder.length() > 0) {
            builder.append(", ");
        }
        builder.append(s);
    }
    return builder.toString();
}
```
使用List，可如下处理：
```java
String join(String... words) {
    return List.of(words)
               .intersperse(", ")
               .foldLeft(new StringBuilder(), StringBuilder::append)
               .toString();
}
```
当然Vavr提供了更简单的方式：
```java
static String join(String... words) {
    return List.of(words).mkString(", ");
}
```
Seq各实现类的时间复杂度如下：

|               head()              | tail()              | get(int)            | update(int, T)      | prepend(T)          | append(T)
------------- | ------------------- | ------------------- | ------------------- | ------------------- | ------------------- | -------------------
Array         | const               | linear              | const               | const               | linear              | linear              
CharSeq       | const               | linear              | const               | linear              | linear              | linear              
Iterator      | const               | const               | --                  | --                  | --                  | --                  
List          | const               | const               | linear              | linear              | const               | linear              
Queue         | const               | const<sup>a</sup>   | linear              | linear              | const               | const               
PriorityQueue | log                 | log                 | --                  | --                  | log                 | log                 
Stream        | const               | const               | linear              | linear              | const<sup>lazy</sup>| const<sup>lazy</sup>
Vector        | const<sup>eff</sup> | const<sup>eff</sup> | const<sup>eff</sup> | const<sup>eff</sup> | const<sup>eff</sup> | const<sup>eff</sup>

标识：
- const — 常数级时间
- const<sup>a</sup> — 分摊常数级时间，少数操作需要更长时间
- const<sup>eff</sup> — 事实（effectively）常数级时间，取决于假设是否成立，比如哈希键值的分布
- const<sup>lazy</sup> — 懒加载常数级时间，操作延后执行
- log — 指数级时间
- linear — 线性时间

Set和Map的接口及实现如图：
![set&map结构](https://github.com/sumuzhou/vavr-notes/blob/master/vavr-overview.png "set&map结构")

Set和Map原理上并没有真正的区别，无非Set目标是值，Map目标是键值对。Vavr的HashMap实现基于[Hash Array Mapped Trie (HAMT)](http://lampwww.epfl.ch/papers/idealhashtrees.pdf)。Map并没有使用特殊的Entry类，而是直接用了Tuple2，极大方便数据结构之间交互：
```java
// = HashMap((0, List(2, 4)), (1, List(1, 3)))
List.of(1, 2, 3, 4).groupBy(i -> i % 2);

// = List((a, 0), (b, 1), (c, 2))
List.of('a', 'b', 'c').zipWithIndex();
```
Set和Map各实现类的时间复杂度如下：

|               contains/Key        | add/put             | remove              | min         
------------- | ------------------- | ------------------- | ------------------- | -------------------
HashMap       | const<sup>eff</sup> | const<sup>eff</sup> | const<sup>eff</sup> | linear              
HashSet       | const<sup>eff</sup> | const<sup>eff</sup> | const<sup>eff</sup> | linear              
LinkedHashMap | const<sup>eff</sup> | linear              | linear              | linear              
LinkedHashSet | const<sup>eff</sup> | linear              | linear              | linear              
Tree          | log                 | log                 | log                 | log                 
TreeMap       | log                 | log                 | log                 | log                 
TreeSet       | log                 | log                 | log                 | log                 

标识：
- const<sup>eff</sup> — 事实（effectively）常数级时间，取决于假设是否成立，比如哈希键值的分布
- log — 指数级时间
- linear — 线性时间
