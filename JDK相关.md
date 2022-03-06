## String类

### 基础

#### 成员变量

```
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {

    private final char value[];

    private int hash; // Default to 0

    private static final long serialVersionUID = -6849794470754667710L;

    private static final ObjectStreamField[] serialPersistentFields =
        new ObjectStreamField[0];
```

#### String被设置为不可变的原因

- 主要是为了“效率” 和 “安全性” 的缘故。若 String允许被继承, 由于它的高度被使用率, 可能会降低程序的性能，所以String被定义成final。
- 由于字符串常量池的存在，为了更有效的管理和优化字符串常量池里的对象，将String设计为不可变性。
- 安全性考虑。因为使用字符串的场景非常多，设计成不可变可以有效的防止字符串被有意或者无意的篡改。
- 作为HashMap、HashTable等hash型数据key的必要。因为不可变的设计，jvm底层很容易在缓存String对象的时候缓存其hashcode，这样在执行效率上会大大提升。

### 深入

#### 字符串常量池

- 字符串常量池在每个VM中只有一份，存放的是字符串常量的引用值。
- 字符串常量池——string pool，也叫做string literal pool。
- 字符串池里的内容是在类加载完成，经过验证，准备阶段之后在堆中生成字符串对象实例，然后将该字符串对象实例的引用值存到string pool中。
- string pool中存的是引用值而不是具体的实例对象，具体的实例对象是在堆中开辟的一块空间存放的。

#### String与Java内存区域

> 首先String name = "abc" 和 String name2 = new String("abc")两者不相等。

使用new 方式创建字符串。首先会在堆上创建一个对象，然后判断字符串常量池中是否存在字符串的常量，如果不存在则在字符串常量池上创建常量；如果没有则不作任何操作。所以name是指向字符串常量池中的常量，而name2是指向堆中的对象，所以name == name2为false。

执行反编译

> ```
> javap -c TestString
> ```

```
0: aload_0       // 表示对this进行操作，把this装在到操作数栈中
1: invokespecial #1        // 调用<init>

0: ldc   #2      //将常量池中的bruis值加载到虚拟机栈中
2: astore_1      //将0中的引用赋值给第一个局部变量，即String name="bruis"
3: ldc  #2       //将常量池中的bruis值加载到虚拟机栈中
5: astore_2      //将3中的引用赋值给第二个局部变量，即String name2= "bruis"
6: new           //调用new指令，创建一个新的String对象，并存入堆中。因为常量池中已经存在了"bruis"，所以新创建的对象指向常量池中的"bruis"
9: dup           //复制引用并并压入虚拟机栈中
10: ldc          //加载常量池中的"bruis"到虚拟机栈中
12: invokespecial //调用String类的构造方法
15: astore_3      //将引用赋值给第三个局部变量，即String name3=new String("bruis")
```

首先执行无参构造函数

这里有一个需要注意的地方，在java中使用"+"连接符时，一定要注意到"+"的连接符效率非常低下，因为"+"连接符的原理就是通过StringBuilder.append()来实现的。所以如：String name = "a" + "b";在底层是先new 出一个StringBuilder对象，然后再调用该对象的append()方法来实现的，调用过程等同于：

> ```
> // String name = "a" + "b";
> String name = new StringBuilder().append("a").append("b").toString();
> ```

#### String的intern方法

字符串常量池由String独自维护，当调用intern()方法时，如果字符串常量池中包含该字符串，则直接返回字符串常量池中的字符串。否则将此String对象添加到字符串常量池中，并返回对此String对象的引用。

#####  重新理解使用new和字面量创建字符串的两种方式

- 使用字面量的方式创建字符串，要分两种情况。

  1. 如果字符串常量池中没有值，则直接创建字符串，并将值存入字符串常量池中；

     > String name = "abc";

     对于字面量形式创建出来的字符串，**JVM会在编译期时对其进行优化并将字面量值存放在字符串常量池中。运行期在虚拟机栈栈帧中的局部变量表里创建一个name局部变量，然后指向字符串常量池中的值。**

     如果字符常量池中存在字面量值，此时要看这个是真正的**字符串值**还是**引用**。如果是字符串值则将局部变量指向常量池中的值；否则指向引用指向的地方。比如常量池中的值是指向堆中的引用，则name变量为将指向堆中的引用

- 使用new的方式创建字符串

  首先在堆中new出一个对象，然后常量池中创建一个指向堆中常量的引用。

```
       /**
        * 首先对于new出的两个String()对象进行字符串连接操作，编译器无法进行优化，只有等到运行期期间，通过各自的new操作创建出对象之后，然后使用"+"连接符拼接字符串，再从字符串常量池中创建三个分别指向堆中"AA"、"BB"，而"AABB"是直接在池中创建的字面量值，这一点可以通过类的反编译来证明，这里就不具体展开了。
		*/
        String a7 = new String("AA") + new String("BB");
        System.out.println("a7 == a7.intern() " + (a7 == a7.intern())); //true

		
		/**
		*  对于下面的实例，首先在编译期就是将"ABABCDCD"存入字符串常量池中，其对于"ABABCDCD"存入的是具体的字面量值，而不是引用。
		*  因为在编译器在编译期无法进行new 操作，所以就无法知道a8的地址，在运行期期间，使用a8.intern()可以返回字符串常量池的字面量。而a9
		*  在编译期经过编译器的优化，a9变量会指向字符串常量池中的"ABABCDCD"。所以a8 == a8.intern()为false；a8 == a9为false；a9 == a8.intern()为
		*  true。
		*/
        String test = "ABABCDCD";
        String a8 = new String("ABAB") + new String("CDCD");
        String a9 = "ABAB" + "CDCD";
        System.out.println("a8 == a8.intern() " + (a8 == a8.intern())); //false
        System.out.println("a8 == a9 " + (a8 == a9)); //false
        System.out.println("a9 == a8.intern() " + (a9 == a8.intern())); //true
```

总结：如果是字面量的话，jvm会在编译器优化将字符串字面量存入到常量池中，intern（）会判断常量池中是否存在字面量，存在则返回常量池的字面量，否则用new创建的话将此String对象添加到字符串常量池中，并返回对此String对象的引用。

编译器优化：

1. 常量可以被认为运行时不可改变，所以编译时被以常量折叠方式优化。
2. 变量和动态生成的常量必须在运行时确定值，所以不能在编译期折叠优化。

#### String类中各种方法

##### equals

```
public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```

equals方法比较是"字符串对象的地址"，如果不相同则比较字符串的内容，实际也就是char数组的内容。

##### hascode

```
    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
```

在String类中，有个字段hash存储着String的哈希值，如果字符串为空，则hash的值为0。String类中的hashCode计算方法就是以31为权，每一位为字符的ASCII值进行运算，用自然溢出来等效取模，经过第一次的hashcode计算之后，属性hash就会赋哈希值。

##### String的hashcode()和equals()

由上面的介绍，可以知道String的equals()方法实际比较的是两个字符串的内容，而String的hashCode()方法比较的是字符串的hash值，那么单纯的a.equals(b)为true，就可以断定a字符串等于b字符串了吗？或者单纯的a.hash == b.hash为true，就可以断定a字符串等于b字符串了吗？答案是否定的。 

在Java中定义了关于hashCode()和equals()方法的规范，总结：

1. 如果两个对象equals()，则它们的hashcode一定相等。
2. 如果两个对象不equals()，它们的hashcode可能相等。
3. 如果两个对象的hashcode相等，则它们不一定equals。
4. 如果两个对象的hashcode不相等，则它们一定不equals。

##### compareTo()

```
    public int compareTo(String anotherString) {
        int len1 = value.length;
        int len2 = anotherString.value.length;
        int lim = Math.min(len1, len2);
        char v1[] = value;
        char v2[] = anotherString.value;

        int k = 0;
        while (k < lim) {
            char c1 = v1[k];
            char c2 = v2[k];
            if (c1 != c2) {
                return c1 - c2;
            }
            k++;
        }
        return len1 - len2;
    }
```

从compareTo()的源码可知，这方法时先比较两个字符串内的字符串数组的ASCII值，如果最小字符串都比较完了都还是相等的，则返回字符串长度的差值；否则在最小字符串比较完之前，字符不相等，则返回不相等字符的ASCII值差值。

##### startsWith(String prefix)

```
    public boolean startsWith(String prefix) {
        return startsWith(prefix, 0);
    }
    
    public boolean startsWith(String prefix, int toffset) {
        char ta[] = value;
        int to = toffset;
        char pa[] = prefix.value;
        int po = 0;
        int pc = prefix.value.length;
        // Note: toffset might be near -1>>>1.
        if ((toffset < 0) || (toffset > value.length - pc)) {
            return false;
        }
        while (--pc >= 0) {
            if (ta[to++] != pa[po++]) {
                return false;
            }
        }
        return true;
    }
```

如果参数字符序列是该字符串字符序列的前缀，则返回true；否则返回false；

##### endsWith(String suffix)

```
public boolean endsWith(String suffix) {
        return startsWith(suffix, value.length - suffix.value.length);
    }
```

其实endsWith()方法就是服用了startsWith()方法而已，传进的toffset参数值时value和suffix长度差值。

##### indexOf(int ch)

```
public int indexOf(int ch) {
        return indexOf(ch, 0);
    }
    
    public int indexOf(int ch, int fromIndex) {
        final int max = value.length;
        if (fromIndex < 0) {
            fromIndex = 0;
        } else if (fromIndex >= max) {
            // Note: fromIndex might be near -1>>>1.
            return -1;
        }

        if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
            // handle most cases here (ch is a BMP code point or a
            // negative value (invalid code point))
            final char[] value = this.value;
            for (int i = fromIndex; i < max; i++) {
                if (value[i] == ch) {
                    return i;
                }
            }
            return -1;
        } else {
            return indexOfSupplementary(ch, fromIndex);
        }
    }
```

对于String的indexOf(int ch)方法，查看其源码可知其方法入参为ASCII码值，然后和目标字符串的ASCII值来进行比较的。其中常量Character.MIN_SUPPLEMENTARY_CODE_POINT表示的是0x010000——十六进制的010000，十进制的值为65536，这个值表示的是十六进制的最大值。

```
private int indexOfSupplementary(int ch, int fromIndex) {
        if (Character.isValidCodePoint(ch)) {
            final char[] value = this.value;
            final char hi = Character.highSurrogate(ch);
            final char lo = Character.lowSurrogate(ch);
            final int max = value.length - 1;
            for (int i = fromIndex; i < max; i++) {
                if (value[i] == hi && value[i + 1] == lo) {
                    return i;
                }
            }
        }
        return -1;
    }
    
    //用于确定指定代码点是否是一个有效的Unicode代码点
    //表达的就是判断codePoint是否在MIN_CODE_POINT和MAX_CODE_POINT值之间，如果是则返回true，否则返回false。
    public static boolean isValidCodePoint(int codePoint) {
        // Optimized form of:
        //     codePoint >= MIN_CODE_POINT && codePoint <= MAX_CODE_POINT
        int plane = codePoint >>> 16;
        return plane < ((MAX_CODE_POINT + 1) >>> 16);
    }
```

java中特意对超过两个字节的字符进行了处理，例如emoji之类的字符。处理逻辑就在indexOfSupplementary(int ch, int fromIndex)方法里。

##### split(String regex, int limit)

```
     public String[] split(String regex, int limit) {
        char ch = 0;
        //if判断中第一个括号先判断一个字符的情况，并且这个字符不是任何特殊的正则表达式。如果要根据特殊字符来截取字符串，则需要使         //用\\来进行字符转义。
        //第二个括号判断有两个字符的情况，并且如果这两个字符是以\开头的，并且不是字母或者数字的时候。
        //第三个括号判断，判断是否是两字节的unicode字符。
        
        if (((regex.value.length == 1 &&
             ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
             (regex.length() == 2 &&
              regex.charAt(0) == '\\' &&
              (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
              ((ch-'a')|('z'-ch)) < 0 &&
              ((ch-'A')|('Z'-ch)) < 0)) &&
            (ch < Character.MIN_HIGH_SURROGATE ||
             ch > Character.MAX_LOW_SURROGATE))
        {
            int off = 0;
            int next = 0;
            // 如果limit > 0，则limited为true
            boolean limited = limit > 0;
            ArrayList<String> list = new ArrayList<>();
            while ((next = indexOf(ch, off)) != -1) {
                if (!limited || list.size() < limit - 1) {
                    list.add(substring(off, next));
                    off = next + 1;
                } else {    // last one
                    // limit > 0，直接返回原字符串
                    list.add(substring(off, value.length));
                    off = value.length;
                    break;
                }
            }
            // 如果没匹配到，则返回原字符串
            if (off == 0)
                return new String[]{this};

            // 添加剩余的字字符串
            if (!limited || list.size() < limit)
                list.add(substring(off, value.length));

            int resultSize = list.size();
            if (limit == 0) {
                while (resultSize > 0 && list.get(resultSize - 1).length() == 0) {
                    resultSize--;
                }
            }
            String[] result = new String[resultSize];
            return list.subList(0, resultSize).toArray(result);
        }
        return Pattern.compile(regex).split(this, limit);
    }
```

对于入参limit：

1. limit > 0，split()方法最多把字符串拆分成limit个部分。
2. limit = 0，split()方法会拆分匹配到的最后一位regex。
3. limit < 0，split()方法会根据regex匹配到的最后一位，如果最后一位为regex，则多添加一位空字符串；如果不是则添加regex到字符串末尾的子字符串。

## CompletableFuture异步编排

CompletableFuture是JDK8中的新特性，主要用于对JDK5中加入的Future的补充。CompletableFuture实现了CompletionStage和Future接口。

1. Concurrent包中的Future在获取结果时会发生阻塞，而CompletableFuture则不会，它可以通过触发异步方法来获取结果。
2. 在CompletableFuture中，如果没有显示指定的Executor的参数，则会调用默认的ForkJoinPool.commonPool()。
3. 调用CompletableFuture的cancel()方法和调用completeExceptionally()方法的效果一样。

在JDK5中，使用Future来获取结果时都非常的不方便，只能通过get()方法阻塞线程或者通过轮询isDone()的方式来获取任务结果，这种阻塞或轮询的方式会无畏的消耗CPU资源，而且还不能及时的获取任务结果，因此JDK8中提供了CompletableFuture来实现异步的获取任务结果。

在Java8中，新增的ForkJoinPool.commonPool()方法，这个方法可以获得一个公共的ForkJoin线程池，这个公共线程池中的所有线程都是Daemon线程，意味着如果主线程退出，这些线程无论是否执行完毕，都会退出系统。

### 源码分析

CompletableFuture类实现了Future接口和CompletionStage接口，Future大家都经常遇到，但是这个CompletionStage接口就有点陌生了，这里的CompletionStage实际上是一个任务执行的一个“阶段”。

```
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
	volatile Object result;       // CompletableFuture的结果值或者是一个异常的报装对象AltResult
    volatile Completion stack;    // 依赖操作栈的栈顶
    ...
    // CompletableFuture的方法
    ... 
	// Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long RESULT;
    private static final long STACK;
    private static final long NEXT;
    static {
        try {
            final sun.misc.Unsafe u;
            UNSAFE = u = sun.misc.Unsafe.getUnsafe();
            Class<?> k = CompletableFuture.class;
            RESULT = u.objectFieldOffset(k.getDeclaredField("result")); //计算result属性的位偏移量
            STACK = u.objectFieldOffset(k.getDeclaredField("stack")); //计算stack属性的位偏移量
            NEXT = u.objectFieldOffset 
                (Completion.class.getDeclaredField("next"));  //计算next属性的位偏移量
        } catch (Exception x) {
            throw new Error(x);
        }
    }
}
```

在CompletableFuture中有一个静态代码块，在CompletableFuture类初始化之前就进行调用，代码块里的内容就是通过Unsafe类去获取CompletableFuture的result、stack和next属性的“偏移量”，这个偏移量主要用于Unsafe的CAS操作时进行位移量的比较。

**runAsync(Runnable, Executor) & runAsync(Runnable)** runAsync()做的事情就是异步的执行任务，返回的是CompletableFuture对象，不过CompletableFuture对象不包含结果。runAsync()方法有两个重载方法，这两个重载方法的区别是Executor可以指定为自己想要使用的线程池，而runAsync(Runnable)则使用的是ForkJoinPool.commonPool()。

```
public static CompletableFuture<Void> runAsync(Runnable runnable) {
        return asyncRunStage(asyncPool, runnable);
    }
    
    private static final boolean useCommonPool =
        (ForkJoinPool.getCommonPoolParallelism() > 1); // 并行级别
private static final Executor asyncPool = useCommonPool ?  
	ForkJoinPool.commonPool() : new ThreadPerTaskExecutor(); //静态常量
	
	static CompletableFuture<Void> asyncRunStage(Executor e, Runnable f) {
        if (f == null) throw new NullPointerException();
        CompletableFuture<Void> d = new CompletableFuture<Void>();
        e.execute(new AsyncRun(d, f));
        return d;
    }
    
    看到asyncRunStage()源码，可以知道任务是由Executor来执行的，那么可想而知Async类一定是实现了Callable接口或者继承了Runnable类，查看Async类：
    static final class AsyncRun extends ForkJoinTask<Void>
            implements Runnable, AsynchronousCompletionTask {
        CompletableFuture<Void> dep; Runnable fn;
        AsyncRun(CompletableFuture<Void> dep, Runnable fn) {
            this.dep = dep; this.fn = fn;
        }

        public final Void getRawResult() { return null; }
        public final void setRawResult(Void v) {}
        public final boolean exec() { run(); return true; }

        public void run() {
            CompletableFuture<Void> d; Runnable f;
            if ((d = dep) != null && (f = fn) != null) {
                dep = null; fn = null;
                if (d.result == null) {
                    try {
                        f.run();
                        d.completeNull();
                    } catch (Throwable ex) {
                        d.completeThrowable(ex);
                    }
                }
                d.postComplete();
            }
        }
    }
    
    final boolean completeNull() {
        return UNSAFE.compareAndSwapObject(this, RESULT, null,
                                           NIL);
    }
```

调用了f.run()之后，然后就是completeNull()方法了，改方法底层通过调用UNSAFE类的compareAndSwapObject()方法，来以CAS的方式将CompletableFuture的结果赋为null。postComplete()就是任务结束后，会执行所有依赖此任务的其他任务，这些任务以一个无锁并发栈的形式存在。 postComplete()的源码还是有点复杂的，先看Completion抽象类。

##### **Completion**

```
abstract static class Completion extends ForkJoinTask<Void>
    implements Runnable, AsynchronousCompletionTask {
    volatile Completion next;      // Treiber stack link

    /**
     * Performs completion action if triggered, returning a
     * dependent that may need propagation, if one exists.
     *
     * @param mode SYNC, ASYNC, or NESTED
     */
    abstract CompletableFuture<?> tryFire(int mode);

    /** Returns true if possibly still triggerable. Used by cleanStack. */
    abstract boolean isLive();

    public final void run()                { tryFire(ASYNC); }
    public final boolean exec()            { tryFire(ASYNC); return true; }
    public final Void getRawResult()       { return null; }
    public final void setRawResult(Void v) {}
}
```

Completion的抽象方法和成员方法的实现逻辑都短短一行或者没有，可以猜到这些方法的实现都是在其子类中。其实现类包括了UniCompletion、BiCompletion、UniAccept、BiAccept等.

而Completion类中还有一个非常重要的成员属性

```
volatile Completion next;
```

有印象的读者应该能记得，CompletableFuture中有一个属性——stack，就是Completion类的。

```
volatile Completion stack;
```

所以CompletableFuture其实就是一个链表的一个数据结构。

```
     abstract static class UniCompletion<T,V> extends Completion {
        Executor executor;                 // executor to use (null if none)
        CompletableFuture<V> dep;          // 代表的依赖的CompletableFuture
        CompletableFuture<T> src;          // 代表的是源CompletableFuture

        UniCompletion(Executor executor, CompletableFuture<V> dep,
                      CompletableFuture<T> src) {
            this.executor = executor; this.dep = dep; this.src = src;
        }
        
        /**
         * 确保当前Completion可以被调用；并且使用ForkJoinPool标记为来确保只有一个线程可以调用，
         * 如果是异步的，则在任务启动之后通过tryFire来进行调用。tryFire方法时在UniAccept类中。
         */
        final boolean claim() {
            Executor e = executor;
            if (compareAndSetForkJoinTaskTag((short)0, (short)1)) {
                if (e == null)
                    return true;
                executor = null; // disable
                e.execute(this);
            }
            return false;
        }

        final boolean isLive() { return dep != null; }
    }
```

claim方法要在执行action前调用，若claim方法返回false，则不能调用action，原则上要保证action只执行一次。

```
static final class UniAccept<T> extends UniCompletion<T,Void> {
        Consumer<? super T> fn;
        UniAccept(Executor executor, CompletableFuture<Void> dep,
                  CompletableFuture<T> src, Consumer<? super T> fn) {
            super(executor, dep, src); this.fn = fn;
        }
        /**
         * 尝试去调用当前任务。uniAccept()方法为核心逻辑。
         */
        final CompletableFuture<Void> tryFire(int mode) {
            CompletableFuture<Void> d; CompletableFuture<T> a;
            if ((d = dep) == null ||
                !d.uniAccept(a = src, fn, mode > 0 ? null : this))
                return null;
            dep = null; src = null; fn = null;
            return d.postFire(a, mode);
        }
    }
    
    final <S> boolean uniAccept(CompletableFuture<S> a,
                                Consumer<? super S> f, UniAccept<S> c) {
        Object r; Throwable x;
        if (a == null || (r = a.result) == null || f == null) //判断源任务是否已经完成了，a表示的就是源任务，a.result就代表的是原任务的结果。
            return false;
        tryComplete: if (result == null) {
            if (r instanceof AltResult) {
                if ((x = ((AltResult)r).ex) != null) {
                    completeThrowable(x, r);
                    break tryComplete;
                }
                r = null;
            }
            try {
                if (c != null && !c.claim())
                    return false;
                @SuppressWarnings("unchecked") S s = (S) r;
                f.accept(s);  //去调用Comsumer
                completeNull();
            } catch (Throwable ex) {
                completeThrowable(ex);
            }
        }
        return true;
    }
```

对于Completion的执行，还有几个关键的属性：

```
static final int SYNC   =  0;//同步
static final int ASYNC  =  1;//异步
static final int NESTED = -1;//嵌套
```

##### CompletionStage

字面意思可以理解为“完成动作的一个阶段”，查看官方注释文档：CompletionStage是一个可能执行异步计算的“阶段”，这个阶段会在另一个CompletionStage完成时调用去执行动作或者计算，一个CompletionStage会以正常完成或者中断的形式“完成”，并且它的“完成”会触发其他依赖的CompletionStage。CompletionStage 接口的方法一般都返回新的CompletionStage，因此构成了链式的调用。

官方定义中，一个Function，Comsumer或者Runnable都会被描述为一个CompletionStage，相关方法比如有apply，accept，run等，这些方法的区别在于它们有些是需要传入参，有些则会产生“结果”。

- Function方法会产生结果
- Comsumer会消耗结果
- Runable既不产生结果也不消耗结果

一个、两个或者任意一个CompletionStage的完成都会触发依赖的CompletionStage的执行，CompletionStage的依赖动作可以由带有then的前缀方法来实现。如果一个Stage被两个Stage的完成给触发，则这个Stage可以通过相应的Combine方法来结合它们的结果，相应的Combine方法包括：thenCombine、thenCombineAsync。但如果一个Stage是被两个Stage中的其中一个触发，则无法去combine它们的结果，因为这个Stage无法确保这个结果是那个与之依赖的Stage返回的结果。

```
	@Test
    public void testCombine() throws Exception {
        String result = CompletableFuture.supplyAsync(() -> {
            return "hello";
        }).thenCombine(CompletableFuture.supplyAsync(() -> {
            return " world";
        }), (s1, s2) -> s1 + " " + s2).join();

        System.out.println(result);
    }
```

虽然Stage之间的依赖关系可以控制触发计算，但是并不能保证任何的顺序。

另外，可以用一下三种的任何一种方式来安排一个新Stage的计算：default execution、default asynchronous execution（方法后缀都带有async）或者custom（自定义一个executor）。默认和异步模式的执行属性由CompletionStage实现而不是此接口指定。

**小结：CompletionStage确保了CompletableFuture能够进行链式调用。**

CompletableFuture的几个核心方法：

**postComplete**

```
final void postComplete() {
        CompletableFuture<?> f = this; Completion h;    //this表示当前的CompletableFuture
        while ((h = f.stack) != null ||                                  //判断stack栈是否为空
               (f != this && (h = (f = this).stack) != null)) {    
            CompletableFuture<?> d; Completion t;      
            if (f.casStack(h, t = h.next)) {                          //通过CAS出栈，
                if (t != null) {
                    if (f != this) {
                        pushStack(h);             //如果f不是this，将刚出栈的h入this的栈顶
                        continue;
                    }
                    h.next = null;    // detach   帮助GC
                }
                f = (d = h.tryFire(NESTED)) == null ? this : d;        //调用tryFire
            }
        }
    }
```

postComplete()方法可以理解为当任务完成之后，调用的一个“后完成”方法，主要用于触发其他依赖任务。

**uniAccept**

```
final <S> boolean uniAccept(CompletableFuture<S> a,
                                Consumer<? super S> f, UniAccept<S> c) {
        Object r; Throwable x;
        if (a == null || (r = a.result) == null || f == null)    //判断当前CompletableFuture是否已完成，如果没完成则返回false；如果完成了则执行下面的逻辑。
            return false;
        tryComplete: if (result == null) {
            if (r instanceof AltResult) {   //判断任务结果是否是AltResult类型
                if ((x = ((AltResult)r).ex) != null) {
                    completeThrowable(x, r);
                    break tryComplete;
                }
                r = null;
            }
            try {
                if (c != null && !c.claim()) //判断当前任务是否可以执行
                    return false;
                @SuppressWarnings("unchecked") S s = (S) r;   //获取任务结果
                f.accept(s);    //执行Comsumer
                completeNull();
            } catch (Throwable ex) {
                completeThrowable(ex);
            }
        }
        return true;
    }
```

uniAccept的入参中，CompletableFuture a表示的是源任务，UniAccept c中报装有依赖的任务，这点需要清除。

**pushStack**

```
	final void pushStack(Completion c) {
        do {} while (!tryPushStack(c));      //使用CAS自旋方式压入栈，避免了加锁竞争
    }

	final boolean tryPushStack(Completion c) {
        Completion h = stack;         
        lazySetNext(c, h);   //将当前stack设置为c的next
        return UNSAFE.compareAndSwapObject(this, STACK, h, c); //尝试把当前栈（h）更新为新值（c）
    }

	static void lazySetNext(Completion c, Completion next) {
        UNSAFE.putOrderedObject(c, NEXT, next);
    }
```

总结：

1. CompletableFuture底层由于借助了魔法类Unsafe的相关CAS方法，除了get或join结果之外，其他方法都实现了无锁操作。
2. CompletableFuture实现了CompletionStage接口，因而具备了链式调用的能力，CompletionStage提供了either、apply、run以及then等相关方法，使得CompletableFuture可以使用各种应用场景。
3. CompletableFuture中有“源任务”和“依赖任务”，“源任务”的完成能够触发“依赖任务”的执行，这里的完成可以是返回正常结果或者是异常。
4. CompletableFuture默认使用ForkJoinPool，也可以使用指定线程池来执行任务。

## ThreadLocal

由JDK提供的一个用于存储每个线程本地副本信息的类，它的编写者就是著名的并发包大神Doug Lea。

官方注释：

```
这个类用于提供线程本地变量，这些变量和普通的变量不同，因为每个线程通过访问ThreadLocal的get或者
是set方法都会有其独立的、初始化的变量副本。ThreadLocal实例通常是希望将线程独有的状态（例如用户ID、交易ID）
线程中的私有静态字段进行关联，即将线程独有的状态存储到线程中。

每个线程都会持有一个指向ThreadLocal变量的隐式引用，只要线程还没有结束，该引用就不会被GC。
但当线程结束后并且其他地方没有对这些副本进行引用，则线程本地实例的所有副本都会被GC。
```

### 适用场景

1. 用于存储线程本地的副本变量，说白了就是为了做到线程隔离。
   - **线程隔离：**在WEB程序中，每个线程就是一个session，不同用户访问程序会通过不同的线程来访问，通过ThreadLocal来确保同一个线程的访问获得的用户信息都是相同的，同时也不会影响其他线程的用户信息。所以ThreadLocal可以很好的确保线程之间的隔离性。
   - **线程资源一致性：**在JDBC内部，会通过ThreadLocal来实现 **线程资源的一致性**。我们都知道，每个HTTP请求都会在WEB程序内部生成一个线程，而每个线程去访问DB的时候，都会从连接池中获取一个Connection连接用于进行数据库交互。那么当一个HTTP请求进来，该请求在程序内部调用了不同的服务，包括搜索服务、下单服务、付款服务等，在这个调用链中每次请求一个服务都需要进行一次数据库交互，那么有一个问题就是如何确保请求过程中和数据库交互的 **事务状态一致** 的问题，如果同一个请求的调用链中connection都不同，则事务就没法控制了，因此在JDBC中通过了ThreadLocal来确保每次的请求都会和同一个connection进行一一对应，确保一次请求链中都用的同一个connection，这就是 **线程资源的一致性**。
   - **线程安全：**基于ThreadLocal存储在Thread中作为本地副本变量的机制，保证每个线程都可以拥有自己的上下文，确保了线程安全。相比于加锁（Synchronize、Lock），ThreadLocal的效率更高。
   - **分布式计算：** 对于分布式计算场景中，即每个线程都计算出结果后，最终通过将ThreadLocal存储的结果取出，并收集。
   - **在SqlSessionManager中的应用：**在SqlSessionManager中，对于SqlSession的存储，就是通过ThreadLocal来进行的。在getConnection()的时候，实际上就是去从ThreadLocal中去获取连接—SqlSession。
   - **在Spring框架中的TransactionContextHolder中的应用：**在淘宝APP中，需要购买某个商品，会涉及交易中台，履约中台。购买一个商品后，会在交易中台去更新订单，同时需要去履约中台进行合约签订。但如果淘宝APP回滚了，则履约中台和交易中台也需要进行业务回滚。对于分布式事务，需要有一个context，即资源上下文，用于存储用户的信息、订单的信息以及来源等，因此在Spring的TransactionContextHolder中，就通过ThreadLocal来存储context。
2. 用于确保线程安全。

### 源码学习

#### ThreadLocal内部使用了哪些数据结构？

ThreadLocal中几个比较重要的数据结构

```
/**
 * 用于ThreadLocal内部ThreadLocalMap数据结构的哈希值，用于降低哈希冲突。
 */
private final int threadLocalHashCode = nextHashCode();

/**
 * 原子操作生成哈希值，初始值为0.
 */
private static AtomicInteger nextHashCode =  new AtomicInteger();

/*
 * 用于进行计算出threadLocalHashCode的哈希值。
 */
private static final int HASH_INCREMENT = 0x61c88647; //1640531527

/**
 * 返回下一个哈希值，让哈希值散列更均匀。
 */
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

**ThreadLocalMap**

```
/**
 * ThreadLocalMap其实就是一个用于ThreadLocal的自定义HashMap，它和HashMap很像。在其内部有一个自定义的Entry类，
 * 并且有一个Entry数组来存储这个类的实例对象。类似于HashMap，ThreadLocalMap同样的拥有初始大小，拥有扩容阈值。
 */
static class ThreadLocalMap {
	/*
	 *  可以看到，Entry类继承了WeakReference类，它的含义是弱引用，即JVM进行GC时，无论当前内存是否够用，
	 *  都会把被WeakReference指向的对象回收掉。
	 */
	static class Entry extends WeakReference<ThreadLocal<?>> {   
	    /** The value associated with this ThreadLocal. */       
	    Object value;
	
	    Entry(ThreadLocal<?> k, Object v) {                      
	        super(k);                                            
	        value = v;                                           
	    }                                                        
	}
	// ThreadLocalMap的初始大小
	private static final int INITIAL_CAPACITY = 16
	                                                
	// 用于存储Entry的数组                                                                               
	private Entry[] table;
	                                      
	private int size = 0;
	
	// 扩容阈值，扩容阈值为初始大小值的三分之二。           
	private int threshold; // Default to 0
	                                    
	private void setThreshold(int len) {          
	    threshold = len * 2 / 3;                  
	}
	                                       
	private static int nextIndex(int i, int len) {
	    return ((i + 1 < len) ? i + 1 : 0);       
	}
	                                     
	private static int prevIndex(int i, int len) {
	    return ((i - 1 >= 0) ? i - 1 : len - 1);  
	}                                                                                                         
}
```

Entry为什么要继承WeakReference，而不是其他的Reference？

1. 是为了在Thread线程在执行过程中，key能够被GC掉，从而在需要彻底GC掉ThreadLocalMap时，只需要调用ThreadLocal的remove方法即可。
2. 如果是用的强引用，虽然Entry到Thread不可达，但是和Value还有强引用的关系，是可达的，所以无法被GC掉。

虽然Entry使用的是WeakReference虚引用，但JVM只是回收掉了ThreadLocalMap中的key，但是value和key是强引用的（value也会引用null），所以value是无法被回收的，所以如果线程执行时间非常长，value持续不GC，就有内存溢出的风险。所以最好的做法就是调用ThreadLocal的remove方法，把ThreadLocal.ThreadLocalMap给清除掉。

#### 源码分析

thread源码中有两个变量

**ThreadLocalMap**变量定义在Thread中，因而每个Thread都拥有自己的ThreadLocalMap变量，互不影响，因而实现了线程隔离性。

还有一个**inheritableThreadLocals**，作用是用于父子线程间ThreadLocal变量的传递。

ThreadLocal的set和get方法

```
// ThreadLocal中的set()方法
	public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
        	// 将当前线程传入，作为ThreadLocalMap的引用，创建出ThreadLocalMap
            createMap(t, value);
    }
    
    // ThreadLocalMap中的set()方法
	private void set(ThreadLocal<?> key, Object value) {
			// 初始化Entry数组
            Entry[] tab = table;
            int len = tab.length;
            // 通过取模计算出索引值
            int i = key.threadLocalHashCode & (len-1);

			// 如果ThreadLocalMap中tab的槽位已经被使用了，则寻找下一个索引位，i=nextIndex(i, len)
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }
				// 如果key引用被回收了，则用新的key-value来替换，并且删除无用的Entry
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            // 清楚哪些get()为空的对象，然后进行rehash。
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

```
public T get() {
		// 获取当前线程
        Thread t = Thread.currentThread();
        // 获取线程t中的ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        // 如果没有获取到ThreadLocalMap，则初始化一个ThreadLocalMap
        return setInitialValue();
    }
	ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    // 初始化
	private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
        	// 把线程存放到当前线程的ThreadLocalMap中
            createMap(t, value);
        return value;
    }
```

知道怎么存储以及获取ThreadLocal之后，还要知道怎么清除ThreadLocal，防止内存泄漏，下面看下remove()源码：

```
// ThreadLocal的remove()方法
	public void remove() {
		// 获取当前线程中的ThreadLocalMap
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }

	// ThreadLocalMap中的remove()方法
	private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            // 通过取模获取出索引位置，
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
               
                    expungeStaleEntry(i);
                    return;
                }
            }
        }

	/**
	 *  清除没用的槽位以及null插槽，并且对其进行重新散列。
	 */
	private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // 将插槽位置的键和值都设置为null
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // 遇到null的插槽，重新散列计算哈希值。
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

#### 总结

 哪些开源框架、源码使用到了ThreadLocal

① JDBC获取Connection相关源码 ② MyBatis中的SqlSessionManager相关源码 ③ Spring框架中的TransactionContextHolder相关源码

关于内存泄漏

由于ThreadLocalMap的Entry继承了WeakReference，所以只要JVM发起了GC，就会回收掉Entry的键，导致当线程持续运行时，ThreadLocal中value值增多，并且没法对其进行GC，所以导致内存泄漏，因此需要调用其remove方法，避免内存泄漏。

## **volatile关键字**

首先并发问题可能会有上下文切换问题，死锁问题。

**上下文切换**

时间片是CPU分给各个线程的时间，因为时间片非常短，所以CPU将会在各个线程之间来回切换从而让用户感觉多个程序是同时执行的。CPU通过时间片分配算法来循环执行任务，因此需要在各个线程之间进行切换。从任务的保存到加载过程就称作“上下文切换”。这里需要知道的是上下文切换是需要系统开销的。

减少上下文切换的措施

1. 无锁并发编程 多线程竞争锁时，会引起上下文切换，所以多线程处理数据时，可以用一些方法来避免使用锁，如将数据的ID按照Hash算法取模分段，不同线程处理不同段的数据。
2. CAS算法 Java的Atomic包使用CAS算法来更新数据，不需要加锁。
3. 使用最少的线程来完成任务 避免不需要的线程。
4. 协程 在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换。

**死锁**

死锁就是两个或者两个以上的线程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象。

死锁产生四个必要条件：①互斥条件②不可抢占条件③请求和保持条件④循环等待条件

避免死锁的方法：①避免获取同一把锁。②避免一个线程在锁内同时占用多个资源，尽量保证每个锁只保持一个资源。③尝试使用定时锁，使用tryLock(timeout)来代替使用内部锁机制。④对于数据库锁，加锁和解锁必须在同一个数据库连接中，否则会出现解锁失败的问题。

**volatile的作用**

1. 是一个轻量级的synchronized。
2. 在多处理器开发中保证的共享变量的“可见性”。
3. 在硬件底层可以禁止指令的重排序。

volatile是如何保证可见性的？

在volatile变量修饰的共享变量进行写操作的时候回多出Lock前缀指令（硬件操作），这个Lock指令在多核处理器下回引发两件事情（硬件操作）：

1. 当前处理器缓存行内的该变量的数据写回到系统内存中。
2. 这个数据写回操作会是其他CPU内缓存内缓存的该变量的数据无效，当处理器对这个数据进行修改操作的时候，会重新从系统内存中读取该数据到处理器缓存里。

Lock引起的将当前处理器缓存该变量的数据写回到系统内存中这一动作，为什么会触发其他CPU缓存行内该变量的数据无效呢？因为变量被修改了，所以其他CPU缓存行内缓存的数据就会无效，但是对于无效了的数据，CPU是怎么操作其变为新的数据呢？这是因为**“缓存一致性协议”**。

**缓存一致性协议**

每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是否过期，当处理器发现自己缓存行对于数据的内存地址被修改了，就会将当前缓存行设置为无效。当处理器对这个数据进行修改操作时，会重新从系统内存中读取该数据到处理器缓存中。

为了实现volatile的内存语义，编译期在生成字节码时会对使用volatile关键字修饰的变量进行处理，在字节码文件里对应位置生成一个Lock前缀指令，Lock前缀指令实际上相当于一个内存屏障（也成内存栅栏），它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成。

**volatile的内存语义**

在了解volatile的内存语义之前，先了解一下happens-before规则。

在JMM规范中，happens-before规则有如下：

1. 程序顺序规则：一个线程内保证语义的串行化
2. volatile规则：volatile变量的写先发生于读，这保证了volatile变量的可见性
3. 锁规则：解锁必定发生于加锁之前
4. 传递性：A先于B，B先于C，A一定先于C

**volatile关键字对于变量的影响**

一个volatile变量的单个读/写操作，与一个普通变量的读/写操作是使用同一个锁来同步，他们之间的执行效果相同。锁的happens-before规则保证释放锁和获取锁的两个线程之间的内存可见性，一个volatile变量的读，总是能够（任意线程）对这个volatile变量最后的写入。可见对于单个volatile的读/写就具有原子性，但如果是多个volatile操作类似于volatile++这种复合操作，就不具备原子性，是线程不安全的操作。

volatile变量的特性：

1. 可见性：对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入
2. 原子性：对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入

**volatile关键字对于线程内存的影响**

从JDK1.5开始，volatile变量的写/读可以实现线程之间通信。从内存语义来说，volatile的读-写与锁的释放-获取有相同的内存效果。**volatile的写与锁的释放有相同的内存语义；volatile的读与锁的获取有相同的内存语义；**

现在有一个线程A和一个线程B拥有同一个volatile变量。当写这个volatile变量时，JMM会把该A线程对应的本地内存中的共享变量值刷新到主内存，当B线程读这个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。这一写一读，达到的就相当于**线程之间通信的效果**。

**volatile内存语义的底层实现原理——内存屏障**

为了实现volatile的内存语义，编译期在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

![image-20210912215646721](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210912215646721.png)

重排序的语义都是通过内存屏障来实现的，那内存屏障是什么呢？硬件层的内存屏障分为两种：Load Barrier 和 Store Barrier即读屏障和写屏障，内存屏障的作用有两个：

- 阻止屏障两侧的的指令重排
- 强制把高速缓存中的数据更新或者写入到主存中。Load Barrier负责更新高速缓存， Store Barrier负责将高速缓冲区的内容写回主存

编译器对所有的CPU来说插入屏障数最小的方案几乎不可能，下面是基于保守策略的JMM内存屏障插入策略：

1. 在每个volatile写操作前面插入StoreStore屏障

2. 在每个volatile写操作后插入StoreLoad屏障

3. 在每个volatile读前面插入一个LoadLoad屏障

4. 在每个volatile读后面插入一个LoadStore屏障

   ![image-20210916101251822](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210916101251822.png)

- StoreStore屏障可以保证在volatile写之前，所有的普通写操作已经对所有处理器可见，StoreStore屏障保障了在volatile写之前所有的普通写操作已经刷新到主存。
- StoreLoad屏障避免volatile写与下面有可能出现的volatile读/写操作重排。因为编译器无法准确判断一个volatile写后面是否需要插入一个StoreLoad屏障（写之后直接就return了，这时其实没必要加StoreLoad屏障），为了能实现volatile的正确内存语意，JVM采取了保守的策略。在每个volatile写之后或每个volatile读之前加上一个StoreLoad屏障，而大多数场景是一个线程写volatile变量多个线程去读volatile变量，同一时刻读的线程数量其实远大于写的线程数量。选择在volatile写后面加入StoreLoad屏障将大大提升执行效率（上面已经说了StoreLoad屏障的开销是很大的）。

![image-20210916110311242](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210916110311242.png)

- LoadLoad屏障保证了volatile读不会与下面的普通读发生重排。
- LoadStore屏障保证了volatile读不回与下面的普通写发生重排。

**组合屏障** 

LoadLoad,StoreStore,LoadStore,StoreLoad实际上是Java对上面两种屏障的组合，来完成一系列的屏障和数据同步功能：

- LoadLoad屏障：对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。
- StoreStore屏障：对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。
- LoadStore屏障：对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。
- StoreLoad屏障：对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。

**volatile的应用场景**

1. 状态标志：多个线程以一个volatile变量作为为状态标志，例如完成**初始化**或者**状态同步**。典型例子AQS的同步状态：
2. 一次性安全发布(最典型的例子就是安全的单例模式，用volatile修饰instance)
3. JDK1.7版本的ConcurrentHashmap的HashEntry的value值，就是通过volatile来修饰的，就是由于volatile的“内存可见性”使得ConcurrentHashMap的get()过程高效、无需加锁。

## 注解

提供了一种安全的类似注释的机制，用来将任何的信息或元数据（metadata）与程序元素（类、方法、成员变量等）进行关联。为程序的元素（类、方法、成员变量）加上更直观更明了的说明，这些说明信息是与程序的业务逻辑无关，并且供指定的工具或框架使用。

注解原理：注解本质是一个继承了Annotation 的特殊接口，其具体实现类是Java 运行时生成的动态代理类。而我们通过反射获取注解时，返回的是Java 运行时生成的动态代理对象$Proxy1。通过代理对象调用自定义注解（接口）的方法，会最终调用AnnotationInvocationHandler 的invoke 方法。该方法会从memberValues 这个Map 中索引出对应的值。而memberValues 的来源是Java 常量池。

四种元注解：

- @Documented – 注解是否将包含在JavaDoc中，一个简单的Annotations 标记注解，表示是否将注解信息添加在java 文档中。
- @Retention – 什么时候使用该注解（定义注解的生命周期）
  1. RetentionPolicy.SOURCE : 在编译阶段丢弃。@Override, @SuppressWarnings都属于这类注解。
  2. RetentionPolicy.CLASS : 在类加载的时候丢弃。（在字节码文件中处理有用，默认使用这种方式）。
  3. RetentionPolicy.RUNTIME : 始终不会丢弃，运行期也保留该注解，因此可以使用反射机制读取该注解的信息。通常用于自定义注解
- @Target – 注解用于什么地方
- @Inherited – 是否允许子类继承该注解，@Inherited 元注解是一个标记注解，@Inherited 阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited 修饰的annotation 类型被用于一个class，则这个annotation 将被用于该class 的子类。

自定义注解规则：

- Annotation 型定义为@interface, 所有的Annotation 会自动继承java.lang.Annotation这一接口,并且不能再去继承别的类或是接口.
- 参数成员只能用public 或默认(default) 这两个访问权修饰
- 参数成员只能用基本类型byte、short、char、int、long、float、double、boolean八种基本数据类型和String、Enum、Class、annotations等数据类型，以及这一些类型的数组.
- 要获取类方法和字段的注解信息，必须通过Java的反射技术来获取 Annotation 对象，因为你除此之外没有别的获取注解对象的方法

## 线程池多余的线程是如何回收的？

ThreadPoolExecutor回收工作线程，一条线程getTask()返回null，就会被回收。

分两种场景。

1. **未调用shutdown() ，RUNNING状态下全部任务执行完成的场景**。

   线程数量大于corePoolSize，线程超时阻塞，超时唤醒后CAS减少工作线程数，如果CAS成功，返回null，线程回收。否则进入下一次循环。当工作者线程数量小于等于corePoolSize，就可以一直阻塞了。

2.  **调用shutdown() ，全部任务执行完成的场景**。shutdown() 会向所有线程发出中断信号，这时有两种可能。

   1. **所有线程都在阻塞**：中断唤醒，进入循环，都符合第一个if判断条件，都返回null，所有线程回收。
   2. **任务还没有完全执行完**：至少会有一条线程被回收。在processWorkerExit(Worker w, boolean completedAbruptly)方法里会调用tryTerminate()，向任意空闲线程发出中断信号。所有被阻塞的线程，最终都会被一个个唤醒，回收。