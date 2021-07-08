[TOC]

# Integer

我们都知道Integer是int的包装类型，那么Integer是怎么实现的呢？

## 1. Integer的类关系

先来看下Integer的类关系

![image-20210512164159889](/Users/colinzwei/Documents/code/person/colinwei91.github.io/Deep In JDK8/数据类型/Untitled.assets/image-20210512164159889.png)

Integer实现了Number类，并且实现了Comparable接口。

先看一下Number类的源码（简化起见，这里的代码把注释去掉了）

```java
public abstract class Number implements java.io.Serializable {

    public abstract int intValue();

    public abstract long longValue();

    public abstract float floatValue();

    public abstract double doubleValue();

    public byte byteValue() {
        return (byte)intValue();
    }

    public short shortValue() {
        return (short)intValue();
    }
}
```

Number类提供了将数值型包装类型拆箱为基础类型的方法。

其中，byteValue()方法和shortValue()方法提供了一种默认实现，通过intValue()方法进行数值类型的强制转换。

其他方法都是抽象方法，必须子类来实现。

Comparable接口提供了用于数值比较的能力，这里暂不详述。

## 2. IntegerCache内部类

不得不提到的一点是，Integer的IntegerCache静态内部类。

从字面上看，很容易理解，Cache即缓存，实际上，它也是起到了一个缓存的作用。

那么问题来了，为什么会有这样一个类呢？它的作用是什么？

### 2.1 IntegerCache的作用

首先解决第一个问题——为什么要有这样一个类？

我们看下IntegerCache的注释：

```java
/**
 * Cache to support the object identity semantics of autoboxing for values between
 * -128 and 127 (inclusive) as required by JLS.
 *
 * The cache is initialized on first usage.  The size of the cache
 * may be controlled by the {@code -XX:AutoBoxCacheMax=<size>} option.
 * During VM initialization, java.lang.Integer.IntegerCache.high property
 * may be set and saved in the private system properties in the
 * sun.misc.VM class.
 */
```

这段注释的意思是：IntegerCache是为了支持JLS中对于-128~127数值的自动装箱的语义规范。它在第一次被使用的时候初始化，并且可以通过VM初始化的变量 -XX:AutoBoxCacheMax=<size>决定java.lang.Integer.IntegerCache.high的值。

那JLS中的规范是怎么描述的呢？下面截取了JLS([The Java® Language Specification](https://docs.oracle.com/javase/specs/jls/se8/html/index.html))中Boxing Conversion部分的相关描述（配上了笔者自己理解的翻译）

```
If the value p being boxed is an integer literal of type int between -128 and 127 inclusive (§3.10.1), or the boolean literal true or false (§3.10.3), or a character literal between '\u0000' and '\u007f' inclusive (§3.10.4), then let a and b be the results of any two boxing conversions of p. It is always the case that a == b.

译：如果P装箱后是介于-128~127（包含127）之间的整型字面量，或者是true或false的布尔字面量，或者是介于'\u0000'和'\u007f'（包含'\u007f'）的字符字面量，如果用a和b分别表示p任意两次装箱的结果，那么a==b要一直成立。

Ideally, boxing a primitive value would always yield an identical reference. In practice, this may not be feasible using existing implementation techniques. The rule above is a pragmatic compromise, requiring that certain common values always be boxed into indistinguishable objects. The implementation may cache these, lazily or eagerly. For other values, the rule disallows any assumptions about the identity of the boxed values on the programmer's part. This allows (but does not require) sharing of some or all of these references. Notice that integer literals of type long are allowed, but not required, to be shared.

译：理想情况下，基础类型的值装箱后的引用要一直相同。但实际中，这在现有的技术条件下不太可行。实际上，该规则退一步讲，就是要求将某些通用的值包装到无法区分的对象中。实现上，可以对这些对象进行提前缓存，或懒缓存。对于其他的值，任何在编程中对于包装类型值的恒等式的假设都是不允许的。该规则允许（但不要求）共享某些或者全部对象的引用。注意，允许（但不要求）对long类型的整型字面量进行共享。
```

JLS的的上述规范要求-128~127（包含127）的装箱类型的值对于"a\==b"永远成立。Java开发者应该都知道，在JAVA中，"\=="运算符的作用是：

 	1. 对于引用类型，比较的是两个引用的地址是否相等；
 	2. 对于基础类型，比较的是值是否相等。

因此，如果没有用这个缓存池的话，就无法满足JLS的规范。

==总结来讲，IntegerCache的作用就是缓存-128~127间的Integer实例。==

实际上，这个特性从 Java 5 开始引入，其范围是固定的 -128~+127。后来在Java 6 后，最大值映射到java.lang.Integer.IntegerCache.high，可以使用 JVM 的启动参数设置最大值。

所以如果在面试的时候，有面试官问你，诸如，a=new Integer(x); b=x; (x>=-128&&x<127)，a==b是true还是false，这类的问题，你可以肯定的回答正确答案。如果他再问你为什么，请把上面的话自信的拿出来，perfect!

### 2.2 IntegerCache的实现

下面我们看下代码是怎么实现的。

```java
    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```

1. IntegerCache类采用的是静态构造方法，所以在第一次被使用的时候初始化。
2. high的取值是Math.max(127, integerCacheHighPropValue)，保证了high>=127，也就保证了JLS的规范。
3. 初始化的时候，将low~high之间的常量，初始化为Integer对象，并缓存到cache数组中。该数组被final修饰，说明，cache数组指向的内存地址是固定的，也就是说这其中的每个Integer对象的地址也是固定的。
4. 在赋值的时候采用的是j=low, j++的方式，说明缓存对象的值包含了-128。

下面我们看Integer类的实现。

Integer类有40+个方法，笔者将其分为以下几类：

1. 构造器方法
2. 重写的Object类的方法
3. 重写的Number类的方法
4. 实现的Comparable接口的compareTo方法
5. 类型转换方法
6. 计算方法

接下来我们一一探究。

## 构造器方法

Integer的构造器有两个，Integer(int) 和 Integer(String)。其中Integer(int)方法就是将参数付给了实例变量，很简单。我们来看下Integer(String)是怎么将字符串转为整型数的。

```java
public Integer(String s) throws NumberFormatException {
    this.value = parseInt(s, 10);
}

public static int parseInt(String s, int radix) throws NumberFormatException
    {
        /*
         * WARNING: This method may be invoked early during VM initialization
         * before IntegerCache is initialized. Care must be taken to not use
         * the valueOf method.
         */

        if (s == null) {
            throw new NumberFormatException("null");
        }

        if (radix < Character.MIN_RADIX) {
            throw new NumberFormatException("radix " + radix +
                                            " less than Character.MIN_RADIX");
        }

        if (radix > Character.MAX_RADIX) {
            throw new NumberFormatException("radix " + radix +
                                            " greater than Character.MAX_RADIX");
        }

        int result = 0;
        boolean negative = false;
        int i = 0, len = s.length();
        int limit = -Integer.MAX_VALUE;
        int multmin;
        int digit;

        if (len > 0) {
            char firstChar = s.charAt(0);
            if (firstChar < '0') { // Possible leading "+" or "-"
                if (firstChar == '-') {
                    negative = true;
                    limit = Integer.MIN_VALUE;
                } else if (firstChar != '+')
                    throw NumberFormatException.forInputString(s);

                if (len == 1) // Cannot have lone "+" or "-"
                    throw NumberFormatException.forInputString(s);
                i++;
            }
            multmin = limit / radix;
            while (i < len) {
                // Accumulating negatively avoids surprises near MAX_VALUE
                digit = Character.digit(s.charAt(i++),radix);
                if (digit < 0) {
                    throw NumberFormatException.forInputString(s);
                }
                if (result < multmin) {
                    throw NumberFormatException.forInputString(s);
                }
                result *= radix;
                if (result < limit + digit) {
                    throw NumberFormatException.forInputString(s);
                }
                result -= digit;
            }
        } else {
            throw NumberFormatException.forInputString(s);
        }
        return negative ? result : -result;
    }
```
