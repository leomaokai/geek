# 编程性能调优

## 高效使用字符串

### String 对象的不可变性

* 保证 String 对象的安全性。假设 String 对象是可变的，那么 String 对象将可能被恶意修改。
* 保证 hash 属性值不会频繁变更，确保了唯一性，使得类似 HashMap 容器才能实现相应的 key-value 缓存功能。
* 可以实现字符串常量池。在 Java 中，通常有两种创建字符串对象的方式，一种是通过字符串常量的方式创建，如 String str=“abc”；另一种是字符串变量通过 new 形式的创建，如 String str = new String(“abc”)。
  * 一种方式创建字符串对象时，JVM 首先会检查该对象是否在字符串常量池中，如果在，就返回该对象引用，否则新的字符串将在常量池中被创建。这种方式可以减少同一个值的字符串对象的重复创建，节约内存。
  * String str = new String(“abc”) 这种方式，首先在编译类文件时，"abc"常量字符串将会放入到常量结构中，在类加载时，“abc"将会在常量池中创建；其次，在调用 new 时，JVM 命令将会调用 String 的构造函数，同时引用常量池中的"abc” 字符串，在堆内存中创建一个 String 对象；最后，str 将引用 String 对象。

> 平常编程时，对一个 String 对象 str 赋值“hello”，然后又让 str 值为“world”，这个时候 str 的值变成了“world”。那么 str 值确实改变了，为什么我还说 String 对象不可变呢？
>
> 这是因为 str 只是 String 对象的引用，并不是对象本身。对象在内存中是一块内存地址，str 则是一个指向该内存地址的引用。所以在刚刚我们说的这个例子中，第一次赋值的时候，创建了一个“hello”对象，str 引用指向“hello”地址；第二次赋值的时候，又重新创建了一个对象“world”，str 引用指向了“world”，但“hello”对象依然存在于内存中。
>
> 也就是说 str 并不是对象，而只是一个对象引用。真正的对象依然还在内存中，没有被改变。

### String 对象的优化

> 如何构建超大字符串？

Java 在进行字符串的拼接时，偏向使用 StringBuilder，这样可以提高程序的效率。

```java
String str = "abcdef";

for(int i=0; i<1000; i++) {
            str = (new StringBuilder(String.valueOf(str))).append(i).toString();
}
```

即使使用 + 号作为字符串的拼接，也一样可以被编译器优化成 StringBuilder 的方式。但再细致些，你会发现在编译器优化的代码中，每次循环都会生成一个新的 StringBuilder 实例，同样也会降低系统的性能。

所以平时做字符串拼接的时候，我建议你还是要显示地使用 String Builder 来提升系统性能。

如果在多线程编程中，String 对象的拼接涉及到线程安全，你可以使用 StringBuffer。但是要注意，由于 StringBuffer 是线程安全的，涉及到锁竞争，所以从性能上来说，要比 StringBuilder 差一些。

### 如何使用 String.intern 节省内存？

```java
String a =new String("abc").intern();
String b = new String("abc").intern();
        
if(a==b) {
    System.out.print("a==b");
}
//a==b
```

在字符串常量中，默认会将对象放入常量池；在字符串变量中，对象是会创建在堆内存中，同时也会在常量池中创建一个字符串对象，String 对象中的 char 数组将会引用常量池中的 char 数组，并返回堆内存对象引用。

如果调用 intern 方法，会去查看字符串常量池中是否有等于该对象的字符串的引用，如果没有，在 JDK1.6 版本中会复制堆中的字符串到常量池中，并返回该字符串引用，堆内存中原有的字符串由于没有引用指向它，将会通过垃圾回收器回收。

在 JDK1.7 版本以后，由于常量池已经合并到了堆中，所以不会再复制具体字符串了，只是会把首次遇到的字符串的引用添加到常量池中；如果有，就返回常量池中的字符串引用。

```markdown
- 在一开始字符串"abc"会在加载类时，在常量池中创建一个字符串对象。
- 创建 a 变量时，调用 new Sting() 会在堆内存中创建一个 String 对象，String 对象中的 char 数组将会引用常量池中字符串。在调用 intern 方法之后，会去常量池中查找是否有等于该字符串对象的引用，有就返回引用。
- 创建 b 变量时，调用 new Sting() 会在堆内存中创建一个 String 对象，String 对象中的 char 数组将会引用常量池中字符串。在调用 intern 方法之后，会去常量池中查找是否有等于该字符串对象的引用，有就返回引用。
- 而在堆内存中的两个对象，由于没有引用指向它，将会被垃圾回收。所以 a 和 b 引用的是同一个对象。
```

![image-20210212132828455](Java性能调优.assets/image-20210212132828455.png)

![image-20210212132835556](Java性能调优.assets/image-20210212132835556.png)

![image-20210212132843374](Java性能调优.assets/image-20210212132843374.png)

> 使用 intern 方法需要注意的一点是，一定要结合实际场景。因为常量池的实现是类似于一个 HashTable 的实现方式，HashTable 存储的数据越大，遍历的时间复杂度就会增加。如果数据过大，会增加整个字符串常量池的负担。

### 如何使用字符串的分割方法？

Split() 方法使用了正则表达式实现了其强大的分割功能，而正则表达式的性能是非常不稳定的，使用不恰当会引起回溯问题，很可能导致 CPU 居高不下。

所以我们应该慎重使用 Split() 方法，我们可以用 String.indexOf() 方法代替 Split() 方法完成字符串的分割。如果实在无法满足需求，你就在使用 Split() 方法时，对回溯问题加以重视就可以了。

## 慎重使用正则表达式

正则表达式是计算机科学的一个概念，很多语言都实现了它。正则表达式使用一些特定的元字符来检索、匹配以及替换符合规则的字符串。

构造正则表达式语法的元字符，由普通字符、标准字符、限定字符（量词）、定位字符（边界字符）组成。

![image-20210212141701650](Java性能调优.assets/image-20210212141701650.png)

### 正则表达式引擎

正则表达式是一个用正则符号写出的公式，程序对这个公式进行语法分析，建立一个语法分析树，再根据这个分析树结合正则表达式的引擎生成执行程序（这个执行程序我们把它称作状态机，也叫状态自动机），用于字符匹配。

目前实现正则表达式引擎的方式有两种：DFA 自动机（Deterministic Final Automaton 确定有限状态自动机）和 NFA 自动机（Non deterministic Finite Automaton 非确定有限状态自动机）。

对比来看，构造 DFA 自动机的代价远大于 NFA 自动机，但 DFA 自动机的执行效率高于 NFA 自动机。

NFA 自动机的优势是支持更多功能。例如，捕获 group、环视、占有优先量词等高级功能。这些功能都是基于子表达式独立进行匹配，因此在编程语言里，使用的正则表达式库都是基于 NFA 实现的。

### NFA 自动机的回溯

用 NFA 自动机实现的比较复杂的正则表达式，在匹配过程中经常会引起回溯问题。大量的回溯会长时间地占用 CPU，从而带来系统性能开销。

```markdown
# 贪婪模式（Greedy）
- 顾名思义，就是在数量匹配中，如果单独使用 +、 ? 、* 或{min,max} 等量词，正则表达式会匹配尽可能多的内容。
# 懒惰模式（Reluctant）
- 在该模式下，正则表达式会尽可能少地重复匹配字符。如果匹配成功，它会继续匹配剩余的字符串。
# 独占模式（Possessive）
- 同贪婪模式一样，独占模式一样会最大限度地匹配更多内容；不同的是，在独占模式下，匹配失败就会结束匹配，不会发生回溯问题。
```

在很多情况下使用懒惰模式和独占模式可以减少回溯的发生

### 正则表达式的优化

* 少用贪婪模式，多用独占模式
* 减少分支选择
* 减少捕获嵌套

## ArrayList还是LinkedList？

ArrayList、Vector、LinkedList 集合类继承了 AbstractList 抽象类，而 AbstractList 实现了 List 接口，同时也继承了 AbstractCollection 抽象类。ArrayList、Vector、LinkedList 又根据自我定位，分别实现了各自的功能。

ArrayList 和 Vector 使用了数组实现，这两者的实现原理差不多，LinkedList 使用了双向链表实现。

### ArrayList 是如何实现的？

> ArrayList 实现类

ArrayList 实现了 List 接口，继承了 AbstractList 抽象类，底层是数组实现的，并且实现了自增扩容数组大小。

ArrayList 还实现了 Cloneable 接口和 Serializable 接口，所以他可以实现克隆和序列化。

ArrayList 还实现了 RandomAccess 接口。RandomAccess 接口是一个标志接口，他标志着“只要实现该接口的 List 类，都能实现快速随机访问”。

> ArrayList 属性

ArrayList 属性主要由数组长度 size、对象数组 elementData、初始化容量 default_capacity 等组成， 其中初始化容量默认大小为 10。

```java
//默认初始化容量
private static final int DEFAULT_CAPACITY = 10;
//对象数组
transient Object[] elementData; 
//数组长度
private int size;
```

transient 关键字修饰该字段则表示该属性不会被序列化，但 ArrayList 其实是实现了序列化接口，这到底是怎么回事呢？

这还得从“ArrayList 是基于数组实现“开始说起，由于 ArrayList 的数组是基于动态扩增的，所以并不是所有被分配的内存空间都存储了数据。

如果采用外部序列化法实现数组的序列化，会序列化整个数组。ArrayList 为了避免这些没有存储数据的内存空间被序列化，内部提供了两个私有方法 writeObject 以及 readObject 来自我完成序列化与反序列化，从而在序列化与反序列化数组时节省了空间和时间。

因此使用 transient 修饰数组，是防止对象数组被其他外部方法序列化。

> ArrayList 构造函数

```java
public ArrayList(int initialCapacity) {
    //初始化容量不为零时，将根据初始化值创建数组大小
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {//初始化容量为零时，使用默认的空数组
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

public ArrayList() {
    //初始化默认为空数组
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

我们在初始化 ArrayList 时，可以通过第一个构造函数合理指定数组初始大小，这样有助于减少数组的扩容次数，从而提高系统性能。

> ArrayList 新增元素

```java
// 直接将元素添加到数组的末尾
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
// 添加元素到任意位置
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}
```

两个方法的相同之处是在添加元素之前，都会先确认容量大小，如果容量够大，就不用进行扩容；如果容量不够大，就会按照原来数组的 1.5 倍大小进行扩容，在扩容之后需要将数组复制到新分配的内存地址。

两个方法也有不同之处，添加元素到任意位置，会导致在该位置后的所有元素都需要重新排列，而将元素添加到数组的末尾，在没有发生扩容的前提下，是不会有元素复制排序过程的。

> ArrayList 删除元素

ArrayList 的删除方法和添加任意位置元素的方法是有些相同的。ArrayList 在每一次有效的删除元素操作之后，都要进行数组的重组，并且删除的元素位置越靠前，数组重组的开销就越大。

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

> ArrayList 遍历元素

ArrayList 是基于数组实现的，所以在获取元素的时候是非常快捷的。

```java
public E get(int index) {
    rangeCheck(index);

    return elementData(index);
}

E elementData(int index) {
    return (E) elementData[index];
}
```

### LinkedList 是如何实现的？

LinkedList 是基于双向链表数据结构实现的，LinkedList 定义了一个 Node 结构，Node 结构中包含了 3 个部分：元素内容 item、前指针 prev 以及后指针 next

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

> LinkedList 实现类

LinkedList 类实现了 List 接口、Deque 接口，同时继承了 AbstractSequentialList 抽象类，LinkedList 既实现了 List 类型又有 Queue 类型的特点；LinkedList 也实现了 Cloneable 和 Serializable 接口，同 ArrayList 一样，可以实现克隆和序列化。

于 LinkedList 存储数据的内存地址是不连续的，而是通过指针来定位不连续地址，因此，LinkedList 不支持随机快速访问，LinkedList 也就不能实现 RandomAccess 接口。

> LinkedList 属性

```java
transient int size = 0;
transient Node<E> first;
transient Node<E> last;
```

这三个属性都被 transient 修饰了，原因很简单，我们在序列化的时候不会只对头尾进行序列化，所以 LinkedList 也是自行实现 readObject 和 writeObject 进行序列化与反序列化。

> LinkedList 新增元素

链表的增加元素，LinkedList 也有添加元素到任意位置的方法

相比 ArrayList 的添加操作来说，LinkedList 的性能优势明显。

> LinkedList 删除元素

在 LinkedList 删除元素的操作中，我们首先要通过循环找到要删除的元素，如果要删除的位置处于 List 的前半段，就从前往后找；若其位置处于后半段，就从后往前找。

这样做的话，无论要删除较为靠前或较为靠后的元素都是非常高效的，但如果 List 拥有大量元素，移除的元素又在 List 的中间段，那效率相对来说会很低。

> LinkedList 遍历元素

LinkedList 的获取元素操作实现跟 LinkedList 的删除元素操作基本类似，通过分前后半段来循环查找到对应的元素。但是通过这种方式来查询元素是非常低效的，特别是在 for 循环遍历的情况下，每一次循环都会去遍历半个 List。

所以在 LinkedList 循环遍历时，我们可以使用 iterator 方式迭代循环，直接拿到我们的元素，而不需要通过循环查找 List。

### 总结

> ArrayList 和 LinkedList 新增元素操作测试(花费时间)

从集合头部位置新增元素：ArrayList>LinkedList

从集合中间位置新增元素：ArrayList<LinkedList

从集合尾部位置新增元素：ArrayList<LinkedList

```markdown
- ArrayList 是数组实现的，而数组是一块连续的内存空间，在添加元素到数组头部的时候，需要对头部以后的数据进行复制重排，所以效率很低；
- ArrayList 在添加元素到数组中间时，同样有部分数据需要复制重排，效率也不是很高；LinkedList 将元素添加到中间位置，是添加元素最低效率的，因为靠近中间位置，在添加元素之前的循环查找是遍历元素最多的操作。
- 而在添加元素到尾部的操作中，我们发现，在没有扩容的情况下，ArrayList 的效率要高于 LinkedList。这是因为 ArrayList 在添加元素到尾部的时候，不需要复制重排数据，效率非常高。而 LinkedList 虽然也不用循环查找元素，但 LinkedList 中多了 new 对象以及变换指针指向对象的过程，所以效率要低于 ArrayList。
```

> ArrayList 和 LinkedList 删除元素操作测试

从集合头部位置删除元素：ArrayList>LinkedList

从集合中间位置删除元素：ArrayList<LinkedList

从集合尾部位置删除元素：ArrayList<LinkedList

> ArrayList 和 LinkedList 遍历元素操作测试

for(;;) 循环：ArrayList<LinkedList

迭代器迭代循环：ArrayList≈LinkedList

```markdown
- LinkedList 的 for 循环性能是最差的，而 ArrayList 的 for 循环性能是最好的。
- 这是因为 LinkedList 基于链表实现的，在使用 for 循环的时候，每一次 for 循环都会去遍历半个 List，所以严重影响了遍历的效率；ArrayList 则是基于数组实现的，并且实现了 RandomAccess 接口标志，意味着 ArrayList 可以实现快速随机访问，所以 for 循环效率非常高。
- LinkedList 的迭代循环遍历和 ArrayList 的迭代循环遍历性能相当，也不会太差，所以在遍历 LinkedList 时，我们要切忌使用 for 循环遍历。
```

> 遍历删除元素时，要使用迭代器

```java
public static void remove(ArrayList<String> list) {
    Iterator<String> it = list.iterator();

    while (it.hasNext()) {
        String str = it.next();

        if (str.equals("b")) {
            it.remove();
        }
    }
}
```

```markdown
- 使用迭代器遍历时，ArrayList 内部创建了一个内部迭代器 iterator，在使用 next() 方法来取下一个元素时，会使用 ArrayList 里保存的一个用来记录 List 修改次数的变量 modCount ，与 iterator 保存了一个 expectedModCount 来表示期望的修改次数进行比较，如果不相等则会抛出异常；
- 而在 foreach 循环中调用 list 中的 remove() 方法，会走到 fastRemove() 方法，该方法不是 iterator 中的方法，而是 ArrayList 中的方法，在该方法只做了modCount++，而没有同步到 expectedModCount。
- 当再次遍历时，会先调用内部类 iteator 中的 hasNext() ,再调用 next() ,在调用 next() 方法时，会对 modCount 和 expectedModCount 进行比较，此时两者不一致，就抛出了 ConcurrentModificationException 异常。
- 所以关键是用 ArrayList 的 remove 还是 iterator 中的 remove 。
```

## Stream如何提高遍历集合效率？

在 Java8 之前，我们通常是通过 for 循环或者 Iterator 迭代来重新排序合并数据，又或者通过重新定义 Collections.sorts 的 Comparator 方法来实现，这两种方式对于大数据量系统来说，效率并不是很理想。

Java8 中添加了一个新的接口类 Stream，他和我们之前接触的字节流概念不太一样，Java8 集合中的 Stream 相当于高级版的 Iterator，他可以通过 Lambda 表达式对集合进行各种非常便利、高效的聚合操作（Aggregate Operation），或者大批量数据操作 (Bulk Data Operation)。

Stream 的聚合操作与数据库 SQL 的聚合操作 sorted、filter、map 等类似。我们在应用层就可以高效地实现类似数据库 SQL 的聚合操作了，而在数据操作方面，Stream 不仅可以通过串行的方式实现数据操作，还可以通过并行的方式处理大批量数据，提高数据的处理效率。

### Demo

```java
// 过滤分组一所中学里身高在 160cm 以上的男女同学

// 传统方式
Map<String, List<Student>> stuMap = new HashMap<String, List<Student>>();
for (Student stu: studentsList) {
    if (stu.getHeight() > 160) { //如果身高大于160
        if (stuMap.get(stu.getSex()) == null) { //该性别还没分类
            List<Student> list = new ArrayList<Student>(); //新建该性别学生的列表
            list.add(stu);//将学生放进去列表
            stuMap.put(stu.getSex(), list);//将列表放到map中
        } else { //该性别分类已存在
            stuMap.get(stu.getSex()).add(stu);//该性别分类已存在，则直接放进去即可
        }
    }
}

// 串行实现
Map<String, List<Student>> stuMap = stuList.stream()
    .filter((Student s) -> s.getHeight() > 160) 
    .collect(Collectors.groupingBy(Student ::getSex)); 
// 并行实现
Map<String, List<Student>> stuMap = stuList.parallelStream()
    .filter((Student s) -> s.getHeight() > 160) 
    .collect(Collectors.groupingBy(Student ::getSex)); 
```

### Stream 如何优化遍历？

> Stream 操作分类

Stream 的操作分类是实现高效迭代大数据集合的重要原因之一

官方将 Stream 中的操作分为两大类：中间操作（Intermediate operations）和终结操作（Terminal operations）。中间操作只对操作进行了记录，即只会返回一个流，不会进行计算操作，而终结操作是实现了计算操作。

中间操作又可以分为无状态（Stateless）与有状态（Stateful）操作，前者是指元素的处理不受之前元素的影响，后者是指该操作只有拿到所有元素之后才能继续下去。

终结操作又可以分为短路（Short-circuiting）与非短路（Unshort-circuiting）操作，前者是指遇到某些符合条件的元素就可以得到最终结果，后者是指必须处理完所有元素才能得到最终结果。

![image-20210212171319858](Java性能调优.assets/image-20210212171319858.png)

我们通常还会将中间操作称为懒操作，也正是由这种懒操作结合终结操作、数据源构成的处理管道（Pipeline），实现了 Stream 的高效。

> Stream 源码实现

![image-20210212171825696](Java性能调优.assets/image-20210212171825696.png)

* BaseStream 和 Stream 为最顶端的接口类。BaseStream 主要定义了流的基本接口方法，例如，spliterator、isParallel 等；Stream 则定义了一些流的常用操作方法，例如，map、filter 等。

* ReferencePipeline 是一个结构类，他通过定义内部类组装了各种操作流。他定义了 Head、StatelessOp、StatefulOp 三个内部类，实现了 BaseStream 与 Stream 的接口方法。

* Sink 接口是定义每个 Stream 操作之间关系的协议，他包含 begin()、end()、cancellationRequested()、accpt() 四个方法。ReferencePipeline 最终会将整个 Stream 流操作组装成一个调用链，而这条调用链上的各个 Stream 操作的上下关系就是通过 Sink 接口协议来定义实现的。

> Stream 操作叠加

一个 Stream 的各个操作是由处理管道组装，并统一完成数据处理的。

管道结构通常是由 ReferencePipeline 类实现的

* Head 类主要用来定义数据源操作，在我们初次调用 names.stream() 方法时，会初次加载 Head 对象，此时为加载数据源操作
* 接着加载的是中间操作，分别为无状态中间操作 StatelessOp 对象和有状态操作 StatefulOp 对象，此时的 Stage 并没有执行，而是通过 AbstractPipeline 生成了一个中间操作 Stage 链表
* 当我们调用终结操作时，会生成一个最终的 Stage，通过这个 Stage 触发之前的中间操作，从最后一个 Stage 开始，递归产生一个 Sink 链

### 总结

* 在循环迭代次数较少的情况下，常规的迭代方式性能反而更好
* 在单核 CPU 服务器配置环境中，也是常规迭代方式更有优势
* 而在大数据循环迭代中，如果服务器是多核 CPU 的情况下，Stream 的并行迭代优势明显

所以我们在平时处理大数据的集合时，应该尽量考虑将应用部署在多核 CPU 环境下，并且使用 Stream 的并行迭代方式进行处理。

## 深入浅出HashMap的设计与优化

HashMap 是基于哈希表的数据结构实现的

哈希表：根据关键码值（Key value）直接进行访问的数据结构。通过把关键码值映射到表中一个位置来访问记录，以加快查找的速度。这个映射函数叫做哈希函数，存放记录的数组就叫做哈希表。

### HashMap 的实现结构

作为最常用的 Map 类，它是基于哈希表实现的，继承了 AbstractMap 并且实现了 Map 接口。

哈希表将键的 Hash 值映射到内存地址，即根据键获取对应的值，并将其存储到内存地址。也就是说 HashMap 是根据键的 Hash 值来决定对应值的存储位置。通过这种索引方式，HashMap 获取数据的速度会非常快。

例如，存储键值对（x，“aa”）时，哈希表会通过哈希函数 f(x) 得到"aa"的实现存储位置。

```markdown
# 解决哈希冲突
- 开放定址法很简单，当发生哈希冲突时，如果哈希表未被装满，说明在哈希表中必然还有空位置，那么可以把 key 存放到冲突位置后面的空位置上去。这种方法存在着很多缺点，例如，查找、扩容等，所以我不建议你作为解决哈希冲突的首选。
- 再哈希法顾名思义就是在同义词产生地址冲突时再计算另一个哈希函数地址，直到冲突不再发生，这种方法不易产生“聚集”，但却增加了计算时间。如果我们不考虑添加元素的时间成本，且对查询元素的要求极高，就可以考虑使用这种算法设计。
- HashMap 则是综合考虑了所有因素，采用链地址法解决哈希冲突问题。这种方法是采用了数组（哈希表）+ 链表的数据结构，当发生哈希冲突时，就用一个链表结构存储相同 Hash 值的数据。
```

### HashMap 的重要属性

HashMap 是由一个 Node 数组构成，每个 Node 包含了一个 key-value 键值对。

Node 类作为 HashMap 中的一个内部类，除了 key、value 两个属性外，还定义了一个 next 指针。当有哈希冲突时，HashMap 会用之前数组当中相同哈希值对应存储的 Node 对象，通过指针指向新增的相同哈希值的 Node 对象的引用。

HashMap 还有两个重要的属性：加载因子（loadFactor）和边界值（threshold）。在初始化 HashMap 时，就会涉及到这两个关键初始化参数。

```markdown
- LoadFactor 属性是用来间接设置 Entry 数组（哈希表）的内存空间大小

# 默认 LoadFactor 值为 0.75。为什么是 0.75 这个值呢？
- 这是因为对于使用链表法的哈希表来说，查找一个元素的平均时间是 O(1+n)，这里的 n 指的是遍历链表的长度，因此加载因子越大，对空间的利用就越充分，这就意味着链表的长度越长，查找效率也就越低。如果设置的加载因子太小，那么哈希表的数据将过于稀疏，对空间造成严重浪费。

- Entry 数组的 Threshold 是通过初始容量和 LoadFactor 计算所得，在初始 HashMap 不设置参数的情况下，默认边界值为 12。如果我们在初始化时，设置的初始化容量较小，HashMap 中 Node 的数量超过边界值，HashMap 就会调用 resize() 方法重新分配 table 数组。这将会导致 HashMap 的数组复制，迁移到另一块内存中去，从而影响 HashMap 的效率。
```

### HashMap 添加元素优化

HashMap 使用 put() 方法添加键值对，当程序将一个 key-value 对添加到 HashMap 中，程序首先会根据该 key 的 hashCode() 返回值，再通过 hash() 方法计算出 hash 值，再通过 putVal 方法中的 (n - 1) & hash 决定该 Node 的存储位置。(尽可能减少哈希冲突)

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
//通过putVal方法中的(n - 1) & hash决定该Node的存储位置
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```

```java
// put 源码
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        //1、判断当table为null或者tab的长度为0时，即table尚未初始化，此时通过resize()方法得到初始化的table
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        //1.1、此处通过（n - 1） & hash 计算出的值作为tab的下标i，并另p表示tab[i]，也就是该链表第一个节点的位置。并判断p是否为null
        tab[i] = newNode(hash, key, value, null);
    //1.1.1、当p为null时，表明tab[i]上没有任何元素，那么接下来就new第一个Node节点，调用newNode方法返回新节点赋值给tab[i]
    else {
        //2.1下面进入p不为null的情况，有三种情况：p为链表节点；p为红黑树节点；p是链表节点但长度为临界长度TREEIFY_THRESHOLD，再插入任何元素就要变成红黑树了。
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            //2.1.1HashMap中判断key相同的条件是key的hash相同，并且符合equals方法。这里判断了p.key是否和插入的key相等，如果相等，则将p的引用赋给e

            e = p;
        else if (p instanceof TreeNode)
            //2.1.2现在开始了第一种情况，p是红黑树节点，那么肯定插入后仍然是红黑树节点，所以我们直接强制转型p后调用TreeNode.putTreeVal方法，返回的引用赋给e
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //2.1.3接下里就是p为链表节点的情形，也就是上述说的另外两类情况：插入后还是链表/插入后转红黑树。另外，上行转型代码也说明了TreeNode是Node的一个子类
            for (int binCount = 0; ; ++binCount) {
                //我们需要一个计数器来计算当前链表的元素个数，并遍历链表，binCount就是这个计数器

                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) 
                        // 插入成功后，要判断是否需要转换为红黑树，因为插入后链表长度加1，而binCount并不包含新节点，所以判断时要将临界阈值减1
                        treeifyBin(tab, hash);
                    //当新长度满足转换条件时，调用treeifyBin方法，将该链表转换为红黑树
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

在 JDK1.8 中，HashMap 引入了红黑树数据结构来提升链表的查询效率。

这是因为链表的长度超过 8 后，红黑树的查询效率要比链表高，所以当链表超过 8 时，HashMap 就会将链表转换为红黑树，这里值得注意的一点是，这时的新增由于存在左旋、右旋效率会降低。

### HashMap 获取元素优化

一旦发生大量的哈希冲突，就会产生 Node 链表，这个时候每次查询元素都可能遍历 Node 链表，从而降低查询数据的性能。

特别是在链表长度过长的情况下，性能将明显降低，红黑树的使用很好地解决了这个问题，使得查询的平均复杂度降低到了 O(log(n))，链表越长，使用黑红树替换后的查询效率提升就越明显。

我们在编码中也可以优化 HashMap 的性能，例如，重写 key 值的 hashCode() 方法，降低哈希冲突，从而减少链表的产生，高效利用哈希表，达到提高性能的效果。

### HashMap 扩容优化

在 JDK1.7 中，HashMap 整个扩容过程就是分别取出数组元素，一般该元素是最后一个放入链表中的元素，然后遍历以该元素为头的单向链表元素，依据每个被遍历元素的 hash 值计算其在新数组中的下标，然后进行交换。这样的扩容方式会将原来哈希冲突的单向链表尾部变成扩容后单向链表的头部。

而在 JDK 1.8 中，HashMap 对扩容操作做了优化。由于扩容数组的长度是 2 倍关系，所以对于假设初始 tableSize = 4 要扩容到 8 来说就是 0100 到 1000 的变化（左移一位就是 2 倍），在扩容中只用判断原来的 hash 值和左移动的一位（newtable 的值）按位与操作是 0 或 1 就行，0 的话索引不变，1 的话索引变成原索引加上扩容前数组。

之所以能通过这种“与运算“来重新分配索引，是因为 hash 值本来就是随机的，而 hash 按位与上 newTable 得到的 0（扩容前的索引位置）和 1（扩容前索引位置加上扩容前数组长度的数值索引处）就是随机的，所以扩容的过程就能把之前哈希冲突的元素再随机分布到不同的索引中去。

### 总结

HashMap 通过哈希表数据结构的形式来存储键值对，这种设计的好处就是查询键值对的效率高。

当查询操作较为频繁时，我们可以适当地减少加载因子

如果对内存利用率要求比较高，我可以适当的增加加载因子。

我们还可以在预知存储数据量的情况下，提前设置初始容量（初始容量 = 预知数据量 / 加载因子）。这样做的好处是可以减少 resize() 操作，提高 HashMap 的效率。

HashMap 还使用了数组 + 链表这两种数据结构相结合的方式实现了链地址法，当有哈希值冲突时，就可以将冲突的键值对链成一个链表。

HashMap 就在 Java8 中使用了红黑树来解决链表过长导致的查询性能下降问题

![image-20210212202425163](Java性能调优.assets/image-20210212202425163.png)

> 实际应用中，我们设置初始容量，一般得是 2 的整数次幂。你知道原因吗？
>
> 2的幂次方减1后每一位都是1，让数组每一个位置都能添加到元素。
> 例如十进制8，对应二进制1000，减1是0111，这样在&hash值使数组每个位置都是可以添加到元素的，如果有一个位置为0，那么无论hash值是多少那一位总是0，例如0101，&hash后第二位总是0，也就是说数组中下标为2的位置总是空的。
> 如果初始化大小设置的不是2的幂次方，hashmap也会调整到比初始化值大且最近的一个2的幂作为capacity。
>
> 减少哈希冲突，均匀分布元素。

## 如何解决高并发下I/O瓶颈？

### 什么是IO

I/O 是机器获取和交换信息的主要渠道，而流是完成 I/O 操作的主要方式。

在计算机中，流是一种信息的转换。流是有序的，因此相对于某一机器或者应用程序而言，我们通常把机器或者应用程序接收外界的信息称为输入流（InputStream），从机器或者应用程序向外输出的信息称为输出流（OutputStream），合称为输入 / 输出流（I/O Streams）。

机器间或程序间在进行信息交换或者数据交换时，总是先将对象或数据转换为某种形式的流，再通过流的传输，到达指定机器或程序后，再将流转换为对象数据。因此，流就可以被看作是一种数据的载体，通过它可以实现数据交换和传输。

### 传统 I/O 的性能问题

> 多次内存复制

输入操作在操作系统中的具体流程

![image-20210212213645283](Java性能调优.assets/image-20210212213645283.png)

* JVM 会发出 read() 系统调用，并通过 read 系统调用向内核发起读请求；
* 内核向硬件发送读指令，并等待读就绪；
* 内核把将要读取的数据复制到指向的内核缓存中；
* 操作系统内核将数据复制到用户空间缓冲区，然后 read 系统调用返回。

在这个过程中，数据先从外部设备复制到内核空间，再从内核空间复制到用户空间，这就发生了两次内存复制操作。这种操作会导致不必要的数据拷贝和上下文切换，从而降低 I/O 的性能。

> 阻塞

一旦发生线程阻塞，这些线程将会不断地抢夺 CPU 资源，从而导致大量的 CPU 上下文切换，增加系统的性能开销。

### 如何优化 I/O 操作

JDK1.4 发布了 java.nio 包（new I/O 的缩写），NIO 的发布优化了内存复制以及阻塞导致的严重性能问题。JDK1.7 又发布了 NIO2，提出了从操作系统层面实现的异步 I/O。

> 使用缓冲区优化读写流操作

传统 I/O 流的实现以字节为单位处理数据。

NIO 与传统 I/O 不同，它是基于块（Block）的，它以块为基本单位处理数据。在 NIO 中，最为重要的两个组件是缓冲区（Buffer）和通道（Channel）。Buffer 是一块连续的内存块，是 NIO 读写数据的中转地。Channel 表示缓冲数据的源头或者目的地，它用于读取缓冲或者写入数据，是访问缓冲的接口。

传统 I/O 和 NIO 的最大区别就是传统 I/O 是面向流，NIO 是面向 Buffer。Buffer 可以将文件一次性读入内存再做后续处理，而传统的方式是边读文件边处理数据。虽然传统 I/O 后面也使用了缓冲块，例如 BufferedInputStream，但仍然不能和 NIO 相媲美。使用 NIO 替代传统 I/O 操作，可以提升系统的整体性能，效果立竿见影。

> 使用 DirectBuffer 减少内存复制

我们知道数据要输出到外部设备，必须先从用户空间复制到内核空间，再复制到输出设备，而在 Java 中，在用户空间中又存在一个拷贝，那就是从 Java 堆内存中拷贝到临时的直接内存中，通过临时的直接内存拷贝到内存空间中去。此时的直接内存和堆内存都是属于用户空间。

NIO 的 Buffer 除了做了缓冲块优化之外，还提供了一个可以直接访问物理内存的类 DirectBuffer。普通的 Buffer 分配的是 JVM 堆内存，而 DirectBuffer 是直接分配物理内存 (非堆内存)。减少了一次数据拷贝。

由于 DirectBuffer 申请的是非 JVM 的物理内存，所以**创建和销毁的代价很高**。DirectBuffer 申请的内存并不是直接由 JVM 负责垃圾回收，但在 DirectBuffer 包装类被回收时，会通过 Java Reference 机制来释放该内存块。

另外一个 Buffer 类：MappedByteBuffer，跟 DirectBuffer 不同的是，MappedByteBuffer 是通过本地类调用 mmap 进行文件内存映射的，map() 系统调用方法会直接将文件从硬盘拷贝到用户空间，只进行一次数据拷贝，从而减少了传统的 read() 方法从硬盘拷贝到内核空间这一步。

> 避免阻塞，优化 I/O 操作

NIO 发布后，通道（Channel）和多路复用器（Selector）这两个基本组件实现了 NIO 的非阻塞

Channel 有自己的处理器，可以完成内核空间和磁盘之间的 I/O 操作。在 NIO 中，我们读取和写入数据都要通过 Channel，由于 Channel 是双向的，所以读、写可以同时进行。

Selector 是 Java NIO 编程的基础。用于检查一个或多个 NIO Channel 的状态是否处于可读、可写。

Selector 是基于事件驱动实现的，我们可以在 Selector 中注册 accpet、read 监听事件，Selector 会不断轮询注册在其上的 Channel，如果某个 Channel 上面发生监听事件，这个 Channel 就处于就绪状态，然后进行 I/O 操作。

一个线程使用一个 Selector，通过轮询的方式，可以监听多个 Channel 上的事件。我们可以在注册 Channel 时设置该通道为非阻塞，当 Channel 上没有 I/O 操作时，该线程就不会一直等待了，而是会不断轮询所有 Channel，从而避免发生阻塞。

目前操作系统的 I/O 多路复用机制都使用了 epoll

>在 JDK1.7 版本中，Java 发布了 NIO 的升级包 NIO2，也就是 AIO。AIO 实现了真正意义上的异步 I/O，它是直接将 I/O 操作交给操作系统进行异步处理。这也是对 I/O 操作的一种优化，那为什么现在很多容器的通信框架都还是使用 NIO 呢？
>
>在Linux中，AIO并未真正使用操作系统所提供的异步I/O，它仍然使用poll或epoll，并将API封装为异步I/O的样子，但是其本质仍然是同步非阻塞I/O
>
>异步I/O模型在Linux内核中没有实现

>一次普通IO需要要进过六次拷贝:
>网卡->内核->临时本地内存->堆内存->临时本地内存->内核->网卡。
>directbfuffer下:
>网卡->内核->本地内存->内核->网卡
>ARP下C直接调用:
>文件->内核->网卡





