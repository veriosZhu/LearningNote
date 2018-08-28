# Java泛型
-----------

## 目录

## 什么是泛型
``` java
List<String> list = new ArrayList<String>();
```
**好处**
1. 类型安全。
2. 消除了代码中许多强制类型转换
3. 为较大优化带来了可能

### 泛型类
``` java
public class Container<K, V> {
    private K key;
    private V value;
}
```

### 泛型接口
如果在泛型类中指定了泛型，可以像一个普通类型一样使用
``` java
public class Container<K, V> {
    private K key;
    ...
    public K getkey() {
        return key;
    }
}
```
如果在类中没有使用泛型，而在定义方法时又想定义类型形参，就会使用以下形式的泛型方法：
``` java
public class Main {
    public static <T> void out(T t) {
        System.out.println(t);
    }
}
```

### 泛型构造器

``` java
public class Person<E> {
    public <T> Person(T t) {
        System.out.println(t);
    }
}

Person<String> a = new <Integer>Person<String>(15); // 注意<>中必须写全类型参数
```

## 类型通配符

`List<?>`这个(?)为通配符，它的元素类型可以匹配任何参数。

>**List<?> 和 List<Object>的区别:**
 List<Object>等价于List，它表示持有任何Object类型的原生List。
 List<?>和List<? extends Object>是不同的，表示具有某种特定类型的非原生List，知识我们不知道那种类型是什么。


以下用法是错误的：(因为无法确定C集合中元素的类型，所以不能向其添加对象)

``` java
List<?> c = new ArrayList<String>();
// 编译器错误
c.add(new Object());
```

### 带限通配符
分为：上限通配符和下限通配符。

上限通配符
``` java
List<? extends Shape> list = new ArrayList<Circle>();
```

下限通配符
``` java
List<? super Circle> list = new ArrayList<Shape>();
```

> 原则：PECS(producer-extends, consumer-super)
 如果参数化类型表示一个T生产者，就用`<? extends T>`；如果它表示一个T消费者，就使用`<? super T>`

 ## 类型擦除
 > 在泛型的内部，无法获得任何有关泛型参数类型的信息。

 注意以下代码的运行结果
 ``` java
 Class c1 = new ArrayList<Integer>().getClass();
 Class c2 = new ArrayList<String>().getClass();
 System.out.println(c1 == c2);
 // 程序输出：true
 ```

 这是因为对于Java来说，它们依然被当成同一类处理，在内存中也只占用一块内存空间。从Java泛型这一概念提出的目的来看，其只是作用于代码编译阶段，对于正确检验泛型结果后，会将泛型的相关信息擦出，也就是说，成功编译过后的class文件中是不包含任何泛型信息的。泛型信息不会进入到运行时阶段。

以下方法可以拿到泛型的类型

 ``` java
 public class GenericTest
{
    private Map<String , Integer> score;
    public static void main(String[] args)
            throws Exception
    {
        Class<GenericTest> clazz = GenericTest.class;
        Field f = clazz.getDeclaredField("score");
// 直接使用getType()取出Field类型只对普通类型的Field有效
        Class<?> a = f.getType();
// 下面将看到仅输出java.util.Map
        System.out.println("score的类型是:" + a);
// 获得Field实例f的泛型类型
        Type gType = f.getGenericType();
// 如果gType类型是ParameterizedType对象
        if(gType instanceof ParameterizedType)
        {
// 强制类型转换
            ParameterizedType pType = (ParameterizedType)gType;
// 获取原始类型
            Type rType = pType.getRawType();
            System.out.println("原始类型是： " + rType);
// 取得泛型类型的泛型参数
            Type[] tArgs = pType.getActualTypeArguments();
            System.out.println("泛型类型是:");
            for (int i = 0; i < tArgs.length; i++)
            {
                System.out.println("第" + i + "个泛型类型是： " + tArgs[i]);
            }
        } else
        {
            System.out.println("获取泛型类型出错！ ");
        }
    }
}
```