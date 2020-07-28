## 1.可见性（Visibility）

下面代码会打印什么值？

```java
public class BadVisibility {
  private static boolean ready;
  private static int number;
  
  private static class ReaderThread extends Thread {
    public void run() {
      while(!ready)
        Thread.yield();
      System.out.println("Number is: " + number);
    }
  }
  
  public static void main(String[] args) {
    new ReaderThread().start();
    number = 42;
    ready = true;
  }
}
```

可能出现的情况：

1. 打印 **`The number is 42`** ，然后程序终止
2. 打印 **`The number is 0`**， 然后程序终止
3. 程序无限循环

**这是因为缓存，也因为处理器允许将不同线程进行重排，除非我们告诉它该怎么处理**

上面程序特点：

- 简单的并发程序：只包含2个线程，2个共享变量
- 程序运行后，会发生什么是不可预见的
- 假设有100个线程和更多的共享变量，应该怎么办呢？

结论：**当数据被不同线程共享时，总是使用合适的同步**



### 1.1 失效数据（Stale data）

- 失效数据的定义：数据可能过时了，在某个读取之后，该数据更新了，但该线程仍然使用旧的数据
- 可以通过对改变对象状态的方法进行同步的方式来避免无效数据



### 1.2 非原子的64位操作（Nonatomic 64-bit operations）

- 安全最低性：表示在线程没有同步的情况下读取变量，可能会得到一个失效值，但至少这个值是由之前某个线程设置的值，而不是一个随机值。
- 这个安全性可应用于所有非64位变量（long & double 类型变量）
- 因为JMM（java内存模型）允许将一个64位读或写操作当做是2个单独的32位操作，如果读和写发生在不同线程中，对非volatile long 或 double 类型变量，有可能一个线程获取到高32位值，而另一个线程获取到低32位值



### 1.3 锁和可见性（Locking and visibility）

- 锁用于保证可见性变量的正确性
- 编译器不会对同步代码块中的操作进行重排序
- 如果有2个线程A & B, 锁可以确保A和B都尝试访问相同锁保护的代码块，如果A先获得锁，A线程中对变量的修改对B线程是可见的（不会产生失效数据）



### 1.4 Volatile 变量（Volatile variables）

- 属性可以使用 `volatile` 关键词进行标记，它可以对变量提供一种弱形式的同步，比如 **`public volatile boolean done`**
- **`public volatile boolean done`** 会告诉编译器和运行时，对 **`done`** 变量的任何操作都不要进行重排序或者缓存
- volatile 变量对于复合操作进行修改变量是没用的，例如 **`a++`** 这种复合操作（它其实是 **读取-修改-更新** 3个操作的复合）
- 因此 **`volatile`** 变量应只用于操作是原子性的变量上，比如实例的boolean标志，比如作为一种完成或者中断或者状态标志的用法

**加锁操作不仅可以确保可见性，还可以确保原子性，而volatile变量只能确保可见性**



> **使用volatile 变量的规则**

只有满足下面条件时，才应该使用 **`volatile`** 变量：

- **对变量的写入不依赖其当前值，或者能保证同一时刻只有一个线程能对线程进行更新**
- 该变量不会和其它状态变量一起纳入不变性条件中
- 在访问变量时不需要加锁



## 2. 发布和逸出（Publication and escape）

**发布**： 指使对象能够在当前作用域之外中的代码中使用

发布方式：

- 存储一个对该对象的引用，这样其余代码就可以找到该对象（比如将对象的引用保存保存到一个公有的静态变量中， **`public static Set<Secret> knownSecrets;`**）
- 在非私有方法中返回该对象
- 作为其它类中的某个方法参数进行传递

有时我们并不想发布某个对象，发布的危害性：

- 发布内部状态变量会破坏封装性，使对象很难保持不变性（invariants）
- 发布未完全构造的对象会破坏线程安全



**逸出（Escape）**： 不小心的发布了某个对象。

有很多种可能导致对象状态变量逸出：

- 将状态变量以 **`public`** 域进行存储
- 存在公有方法返回某个状态变量
- 允许状态变量包含内部类（inner class）



**外部方法（Alien method）**:假设有一个类 **`C`**, 对于C来说，外部方法是指，行为不完全由C规定的方法，包括其他类型中方法以及类C中可以被改写的方法（既不是私有（private）方法，也不是终结（final）方法）。**将某个对象传递给外部方法，意味着不小心将这个对象发布出去了**。

### 2.1 安全的对象构造

**隐式的使 `this` 引用逸出** 示例：

```java
public class ThisEscape {
  public ThisEscape(EventSource source) {
    souce.registerListener(
    	new EventListener() {
        public void onEvent(Event e) {
          doSomething(e);
        }
    	}
    );
  }
}
```



- 当内部类被发布了，包裹类也会被发布（当内部的 `EventListener` 实例被发布时，在外部封装的 **`ThisEscape`** 实例也逸出了）
- 发布一个尚未构造完成的对象
- **规则：不要在构造过程中使 `this` 引用逸出**

改进示例：**如果想在构造函数中注册一个事件监听器或启动线程，那么可以使用一个私有的构造函数和一个公共的工厂方法，从而避免不正确的构造过程**

```java
class SafeListener {
  private final EventListener listener;
  
  private SafeListener() {
    listener = new EventListener() {
      public void onEvent(Event e) {
        doSomething(e);
      }
    };
  }
  
  public static SafeListener newInstance(EventSouce source) {
    SafeListener safe = new SafeListener();
    source.registerListener(safe.listener);
    return safe;
  }
}
```



## 3. 线程封闭（Thread confinement）

- 访问共享，可变数据需要使用同步机制
- **避免同步最简单的方式就是不要在多个线程之间进行数据共享，这就称之为线程封闭**
- 示例
  - Swing框架
  - Connection pooling(连接池)
- java语言本身是没有方式来定义一个对象是否是封闭在某个线程中的，这需要程序员去实现



### 3.1 Ad-hoc 线程封闭

定义：维护线程封闭性的职责完全由程序实现来承担

- 最简单的线程封闭形式
- ad-hoc 线程封闭是非常脆弱的
- 某些时候，单线程子系统提供的简便性要胜过Ad-hoc线程封闭技术的脆弱性
- 应尽可能少的使用ad-hoc，在可能情况下，应该使用更强的线程封闭技术（例如栈封闭或ThreadLocal类）



### 3.2 栈封闭（Stack confinement）

- 在栈封闭中，只能通过局部变量访问对象
- 更易于维护，比ad-hoc封闭更加的健壮
- 简单类型变量（primitive type variables）总是栈封闭的（因为简单类型是值传递而不是引用传递）

例子：

```java
public class TheArk {
  public int loadTheArk(Collection<Animal> candidates) {
    SortedSet<Animal> animals;
    Ark ark = new Ark();
    int numPairs = 0;
    Animal candidate = null;
    
    // animals 封闭在方法中 不要使它逸出
    animals = new TreeSet<Animal>(new SpeciesGenderComparator<Animal>());
    animals.add(candidates);
    for(Animal a: animals) {
      if (candidate == null || !candidate.isPotentialMate(a)) {
        candidate = a;
      } else {
        ark.load(new AnimalPair(candidate, a));
        ++numPairs;
        candidate = null;
      }
    }
    
    return numPairs;
  }
}
```



### 3.3 ThreadLocals

- **ThreadLocals类提供了 `get & set` 方法，这些方法为每个使用该变量的线程都存有一份独立的副本, 因此 `get` 总是返回由当前执行线程在调用set时设置的最新值**
- 可以将其看做为各个线程中的一个本地可变的单例对象
- 提供2个重要方法： **`initialValue() & get() & set()`**
- 要小心的使用ThreadLocal.不该将其用作为全局变脸或者某个方法的隐藏参数，这会影响维护性
- **ThreadLocal对象通常用于防止对可变的单例变量或全局变量进行共享**
- 当某个频繁操作需要一个临时对象，例如一个缓冲区，而又希望避免每次执行时都重新分配该临时对象



## 4.不变性（Immutability）

另一种避免使用同步的方式就是使用 **不可变对象**，即对象状态在创建之后就不会发生变化的对象。

不可变对象特点：**不可变对象总是线程安全的**

- 不存在失效数据
- 不存在可见性问题
- 不用关心原子性
- 不用担心逸出

注意事项：

- 将对象中所有的字段都使用 **`final`** 声明是不足以是对象变为不可变，因为final字段有可能引用一个可变对象
- **当满足以下条件，对象才是不可变对象**
  - 对象状态在创建之后不会发生变化
  - 所有字段都是 **`final`**
  - 对象是正确创建的（在对象创建期间，this引用不会逸出）



> final 域

- final 字段用于辅助创建不可变对象
- 如果一个对象不能完全不可变，但至少可以使用 **`final`** 让大部分对象不可变



> 使用 volatile类型发布不可变对象

**对数值及其因素分解结果进行缓存的不可变容器类**

```java
@Immutable
class OneValueCache {
  private final BigInteger lastNumber;
  private final BigInteger[] lastFactors;
  
  public OneValueCache(BigInteger i, BigInteger[] factors) {
    lastNumber = i;
    lastFactors = Arrays.copyOf(factors, factors.length);
  }
  
  public BigInteger[] getFactors(BigInteger i) {
    if (lastNumber == null || !lastNumber.equals(i)) {
      return null;
    } else {
      return Arrays.copyOf(lastFactors, factors.length);
    }
  }
}
```



## 5.安全发布（Safe Publication）

有时需要发布对象，重要的是如何以线程安全的方式发布对象。

错误示例：

```java
public class HolderClient {
  public Holder holder;
  public void initialise() {
    holder = new Holder(42);
  }
}

class Holder {
  private int n;
  public Holder(int n) {
    this.n = n;
  }
  public void assertSanity() {
    if (n != n) {
      throw new AssertionError("WTF?”)；
    }
  }
}
```

可能出现的问题：

- **由于可见性问题**， 除了发布线程外，其它线程可以看到的Holder引用的值是最新的，因此将看到一个空引用或者之前的旧值
- 更糟糕的是，线程看到Holder引用的值是最新的，但Holder状态的值却是失效的

因为：

- 发布一个包含public 引用的对象是不安全的
- 这里 **`Holder`** 对象自身并不是危险的，而是它发布的方式是危险的



### 5.1 不可变对象和初始化安全性

- JAVA内存模型为不可变对象的共享提供了一种特殊的初始化安全性保证
- 这意味着在发布不可变对象的引用时没有使用同步，也仍然可以安全的访问该对象
- 这也意味着，将上面 **`Holder`** 类中的 **`n`** 声明为 **`public final`**,发布将是安全的



### 5.2 安全发布的最佳实践

可变对象必须通过安全的方式进行发布，这通常意味着在发布和使用该对象的线程时都必须使用同步。

一个正确构造的对象可以通过以下方式安全的发布：

- 在静态初始化函数中初始化一个对象引用
- 将对象的引用保存到 volatile 类型的域或者AtomicReference对象中
- 将对象的引用保存到某个正确构造对象的final类型域中
- 将对象的引用保存到一个由锁保护的域中



> 在容器中安全发布

最后一个是针对容器的安全发布最佳实践。容器类型包含：

- HashTable
- synchronizedList & synchronizedSet & synchronizedMap
- ConcurrentMap
- Vector
- CopyOnWriteArrayList & CopyOnWriteArraySet
- BlockingQueue & ConcurrentQueue

最简单和最安全的发布对象的方式是使用静态初始化器： **`public static Holder holder = new Holder(42);`**



### 5.3 事实不可变对象（Effectively immutable objects）

- 定义：如果对象从技术上来看是可变的，但其状态在发布后不会再改变，那么把这种对象称为 **事实不可变对象**
- 这种类型的对象可以在没有同步的情况下在任何线程中安全的被使用



### 5.4 对象可变性规则（Object mutability rules）

- 不可变对象可以通过任意机制发布
- **事实不可变对象必须安全的发布**
- **可变对象必须安全的发布，要么是线程安全的，要么通过锁来保护其状态**



### 5.5 对象共享策略（Object sharing policies）

当发布一个对象时，必须明确的说明对象访问方式。

在并发程序中使用和共享对象时，可以使用一些实用的策略：

- **线程封闭(Thread-confined)**：线程封闭的对象只能由一个线程拥有，对象被封闭在该线程中，并且只能由这个线程修改
- **只读共享(Shared read-only)**：在没有额外的同步的情况下，共享的只读对象可以由多个线程并发的访问，但任何线程都不能修改它。共享的只读对象包括，不可变对象和事实不可变对象
- **线程安全共享(Shared thread-safe)**：线程安全的对象在其内部实现同步，因此多个线程可以通过对象的公有接口进行访问而不需要进一步同步
- **保护对象(Guarded)**：被保护的对象只有通过持有特定的锁来访问。保护对象包括封装在其它线程安全独享中的对象，以及已发布的并且由特定锁保护的对象。



2020年07月28日18:47:11

