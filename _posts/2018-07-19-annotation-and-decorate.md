---
title: 面向对象编程：Java注解与Python装饰器
category: Java
---
在面向切面编程中，`python`跟`java`有一个写法类似的语法，就是`@`，通过使用这个功能，两者都能达到切面编程的目的，但`python`中叫装饰器，是一种语法糖，而`java`中叫注解，是一种元数据的标注语法。下面来对比一下两者的异同。

## 一.简介
#### 1. 注解简介
在`Java`中@表示注解，单纯标记一个`Java`语言元素。仅提供附加元数据支持，并不能实现任何操作。需要另外的`Scanner`根据元数据执行相应操作。当然用户也可以通过`@interface`来自定义注解。在运行中可以通过`classObject.getAnnotation(x)`来获取注解对象。
#### 2.装饰器简介
仅提供定义劫持，能够对类及其方法的定义并没有提供任何附加元数据的功能。装饰器本质上是一个`Python`函数或类，它可以让其他函数或类在不需要做任何代码修改的前提下增加额外功能，装饰器的返回值也是一个函数/类对象。它经常用于有切面需求的场景，比如：插入日志、性能测试、事务处理、缓存、权限校验等场景。
#### 3.个人理解 
- `Java`的`AOP`建立在反射的基础之上，而`Python`仅仅是直接的一层函数调用。
- `Java`注解仅仅用来存储元数据，本身是用来做标记的，你需要用这个标记干嘛，需要自己实现对应的逻辑实现，而`Python`中的装饰器是一个语法糖，它本身就涉及到一个返回函数的概念，可以说返回函数是装饰器得以实现的基石。

~~~python
@decorator
def function():
    pass
~~~

这个语法糖相当于实现的是

~~~python
def function():
    pass
function = decorator(function)
~~~

## 二.Java注解
同`classs`和`interface`一样，注解也属于一种类型。它是在`Java SE 5.0`版本中开始引入的概念。

### 1.元注解
元注解是可以注解到注解上的注解，或者说元注解是一种基本注解，但是它能够应用到其它的注解上面。它的作用和目的就是给其他普通的标签进行解释说明的。元标签有 `@Retention`、`@Documented`、`@Target`、`@Inherited`、`@Repeatable` 5 种。

#####@Retention
- RetentionPolicy.SOURCE 注解只在源码阶段保留，在编译器进行编译时它将被丢弃忽视。 
- RetentionPolicy.CLASS 注解只被保留到编译进行的时候，它并不会被加载到 JVM 中。 
- RetentionPolicy.RUNTIME 注解可以保留到程序运行的时候，它会被加载进入到 JVM 中，所以在程序运行时可以获取到它们。

##### @Documented
顾名思义，这个元注解肯定是和文档有关。它的作用是能够将注解中的元素包含到 Javadoc 中去。

##### @Target
当一个注解被 @Target 注解时，这个注解就被限定了运用的场景，@Target 有下面的取值
- ElementType.ANNOTATION_TYPE 可以给一个注解进行注解
- ElementType.CONSTRUCTOR 可以给构造方法进行注解
- ElementType.FIELD 可以给属性进行注解
- ElementType.LOCAL_VARIABLE 可以给局部变量进行注解
- ElementType.METHOD 可以给方法进行注解
- ElementType.PACKAGE 可以给一个包进行注解
- ElementType.PARAMETER 可以给一个方法内的参数进行注解
- ElementType.TYPE 可以给一个类型进行注解，比如类、接口、枚举

##### @Inherited
说的比较抽象。代码来解释。

~~~java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@interface Test {}

@Test
public class A {}

public class B extends A {}
~~~

注解`Test`被`@Inherited`修饰，之后类`A`被`Test`注解，类`B`继承`A`,类`B`也拥有`Test`这个注解。
##### @Repeatable
Repeatable 自然是可重复的意思。@Repeatable 是 Java 1.8 才加进来的，所以算是一个新的特性。
什么样的注解会多次应用呢？通常是注解的值可以同时取多个。

~~~java
@interface Persons {
    Person[]  value();
}


@Repeatable(Persons.class)
@interface Person{
    String role default "";
}


@Person(role="artist")
@Person(role="coder")
@Person(role="PM")
public class SuperMan{

}
~~~

### 2.注解的属性
注解的属性也叫做成员变量。注解只有成员变量，没有方法。注解的成员变量在注解的定义中以“无形参的方法”形式来声明，其方法名定义了该成员变量的名字，其返回值定义了该成员变量的类型。

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {
    int id();
    String msg();
}
~~~

在使用的时候，我们应该给它们进行赋值。赋值的方式是在注解的括号内以 value=”” 形式。

~~~java
@TestAnnotation(id=3,msg="hello annotation")
public class Test {
}
~~~

如果一个注解内仅仅只有一个名字为 value 的属性时，应用这个注解时可以直接接属性值填写到括号内。

~~~java
public @interface Check {
    String value();
}

@Check("hi")
int a;
~~~

### 3.预制注解
`@Override`/`@SuppressWarnings`/`@SafeVarargs`/`@FunctionalInterface`，这些都是大家熟悉的，就不细说了。

### 4.自定义注解
举个自定义注解的例子
- 注解使用效果

~~~java
    @SQLString(name = "NAME" , value = 30)
    private String name;
~~~

在这里，通过使用`@FieldTypeAnnotation`，将值指定给`name`
- 注解定义

~~~java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface SQLString {

    //对应数据库表的列名
    String name() default "";

    //列类型分配的长度，如varchar(30)的30
    int value() default 0;

    Constraints constraint() default @Constraints;
}
~~~

- 注解行为注入

~~~java
  public static String createTableSql(String className) {
      ...
      //获取字段上的注解
      Annotation[] anns = field.getDeclaredAnnotations();
      if(anns.length < 1)
        continue; // Not a db table column

      ...
      //判断String类型
      if(anns[0] instanceof SQLString) {
        SQLString sString = (SQLString) anns[0];
        // Use field name if name not specified.
        if(sString.name().length() < 1)
          columnName = field.getName().toUpperCase();
        else
          columnName = sString.name();
        columnDefs.add(columnName + " VARCHAR(" +
                sString.value() + ")" +
                getConstraints(sString.constraint()));
      }

    }
~~~



### 5.AOP之AspectJ
当我们需要在不修改原来业务代码的情况下，做一些切面工作，例如插桩打点，这时候可以用到`AspectJ`，`aspectjx`默认会遍历项目编译后所有的`.class`文件和依赖的第三方库去查找符合织入条件的切点，为了提升编译效率，可以加入过滤条件指定遍历某些库或者不遍历某些库。
举个例子，假设，我需要在`onCreate`之前插桩打`log`，可以直接提供一个切面类即可

~~~java
@Aspect
public class AspectTest {
    final String TAG = AspectTest.class.getSimpleName();

    @Before("execution(* *..MainActivity+.on**(..))")
    public void method(JoinPoint joinPoint) throws Throwable {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        String className = joinPoint.getThis().getClass().getSimpleName();

        Log.e(TAG, "class:" + className);
        Log.e(TAG, "method:" + methodSignature.getName());
    }
}
~~~

当用户运行`App`时，则会输出

~~~bash
E/Aspectest: class: MainActivity
E/Aspectest: method: onCreate
~~~




## 三.Python装饰器

#### 1.返回函数
由于`Python`有一个内建属性`__call__`，这个是一个很神奇的特性，只要某个类型中有`__call__`方法，我们可以把这个类型的对象当作函数来使用。这点是`Java`跟`C++`不一样的地方，Python 中的函数可以像普通变量一样当做参数传递给另外一个函数

~~~python
In [134]: %cpaste
Pasting code; enter '--' alone on the line to stop or use Ctrl-D.
:
def lazy_sum(*args):
    def sum():
        ax = 0
        for n in args:
            ax = ax + n
        return ax
    return sum


In [135]: f = lazy_sum(1,3,5,7,9)

In [136]: f
Out[136]: <function __main__.lazy_sum.<locals>.sum>

In [137]: f()
Out[137]: 25

In [138]: dir(f)
Out[138]: 
['__annotations__',
 '__call__',
 '__class__',
 ...
 '__getattribute__',
...
 '__setattr__',
]
~~~

### 2.无参装饰器

~~~python
'''使用语法糖@来装饰函数，相当于: myfunc = deco(myfunc)'''
 
def deco(func):
    def _deco():
        print("before myfunc() called.")
        func()
        print("  after myfunc() called.")
        # 不需要返回func，实际上应返回原函数的返回值
    return _deco
 
@deco
def myfunc():
    print(" myfunc() called.")
 
myfunc()
~~~

### 3.定参函数装饰器
函数带参数，我们只要把装饰器最内层函数跟调用函数的参数列表保持一致即可。

~~~python
def deco(func):
    def _deco(a, b):
        print("before myfunc() called.")
        ret = func(a, b)
        print("  after myfunc() called. result: %s" % ret)
        return ret
    return _deco
 
@deco
def myfunc(a, b):
    print(" myfunc(%s,%s) called." % (a, b))
    return a + b
 
myfunc(1, 2)
~~~

### 4.多参函数装饰器

~~~python
def deco(func):
    def _deco(*args, **kwargs):
        print("before %s called." % func.__name__)
        ret = func(*args, **kwargs)
        print("  after %s called. result: %s" % (func.__name__, ret))
        return ret
    return _deco
 
@deco
def myfunc(a, b):
    print(" myfunc(%s,%s) called." % (a, b))
    return a+b
 
@deco
def myfunc2(a, b, c):
    print(" myfunc2(%s,%s,%s) called." % (a, b, c))
    return a+b+c
 
myfunc(1, 2)
myfunc2(1, 2, 3)
~~~

### 5.带参装饰器
装饰器带参数，则装饰器函数则变成了三层，我们需要在最外层把装饰器的参数传递进去。

~~~python
def deco(arg):
    def _deco(func):
        def __deco():
            print("before %s called [%s]." % (func.__name__, arg))
            func()
            print("  after %s called [%s]." % (func.__name__, arg))
        return __deco
    return _deco
 
@deco("mymodule")
def myfunc():
    print(" myfunc() called.")
 
@deco("module2")
def myfunc2():
    print(" myfunc2() called.")
 
myfunc()
~~~