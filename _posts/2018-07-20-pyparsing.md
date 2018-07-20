---
title: 使用Pyparsing来定制自己的解析器
category: Python
---

`Pyparsing`是纯`python`的，易于使用。`Pyparsing`提供了一系列类让你可以以单独的表达式元素开始来构建解析器。 其表达式使用直觉的符号组合，如`+`表示将一个表达式加到另一个后面。`|`,`^`表示解析多选 (意为匹配第一个或匹配最长的).表达式的重复可以以类的形式表示，如`OneOrMore`,`ZeroOrMore`,`Optional`.

## 一.`Pyparsing`的应用场景
在工作中，可能经常会遇到下面这些需求
- "我需要解析这个日志文件..."
- "我需要从网页中提取数据..."
- "我们需要一个简单的命令行解释器..."
- "我们的源代码需要移植到新API集上..."
- "我们需要设计自己的DSL"

通常简单点的，可能直接会使用`awk`，复杂点的可能就是结合正则表达式了。但在这里要推荐一种比较小众的方式，使用`Pyparsing`来定制自己的解析器。而上面的场景，都可以用`Pyparsing`来实现。下面一起看看`Pyparsing`是什么。

## 二.`Pyparsing`介绍
那`Pyparsing`是什么呢？简单来说就是解析表达式使用标准的`python`类标记和符号表示。因为它的规则可读性很强，因此很方便维护以及拓展，在中等复杂的需求中，可以拿来使用。 复杂点说，`Pyparsing`是：

- 100%纯`python`,没有的动态链接库(`DLLs`)或者共享库包含其中，所以你可以在`python2.3`能够通过编译的任何地方使用它。
- 解析表达式使用标准的`python`类标记和符号表示。没有单独的代码生成过程也没有特殊符号和标记，这将使得你的应用易于开发，理解和维护。
- 对于常见的模式准备了辅助方法:
    - `C`,`C++`,`Java`,`Python`,`HTML`注释
    - 引号字符串(使用单个或双引号，除了',''转义情况外)
    - `HTML`与`XML`标签(包含上下级以及属性操作)
    - 逗号分隔以及被限制的列表表达式
- 轻量级封装-`Pyparsing`的代码包含在单个`python`文件中，容易放进`site-packages`目录下，或者被你的应用直接包含。
- 宽松的许可证，`MIT`许可证使得你可以随意进行非商用或商业应用。
可能这么说还是比较抽象，直接看下面的例子吧。

## 三.`Pyparsing`使用演示
看一个`traceview`的解析例子，假设原始文件如下：

~~~bash
AsyncTask #3
pool-2-thread-5
pool-3-thread-1
uil-pool-2-thread-1
uil-pool-2-thread-2
uil-pool-2-thread-3
uil-pool-2-thread-4
Trace (threadID action usecs class.method signature):
xit         0 ..dalvik.system.VMDebug.startMethodTracingFilename (Ljava/lang/String;IIZI)V  VMDebug.java
xit         0 ..com.android.org.conscrypt.NativeCrypto.EVP_DigestUpdate (Lcom/android/org/conscrypt/OpenSSLDigestContext;[BII)V NativeCrypto.java
xit       218 .dalvik.system.VMDebug.startMethodTracing (Ljava/lang/String;IIZI)V   VMDebug.java
xit       225 android.os.Debug.startMethodTracing (Ljava/lang/String;II)V   Debug.java
xit       230-android.os.Debug.startMethodTracing (Ljava/lang/String;I)V    Debug.java
xit       266-java.lang.reflect.Method.invoke (Ljava/lang/Object;[Ljava/lang/Object;Z)Ljava/lang/Object;    Method.java
ent       528 ..java.lang.ClassLoader.loadClass (Ljava/lang/String;)Ljava/lang/Class;   ClassLoader.java
ent       543 ...java.lang.ClassLoader.loadClass (Ljava/lang/String;Z)Ljava/lang/Class; ClassLoader.java
ent       548 ....java.lang.ClassLoader.findLoadedClass (Ljava/lang/String;)Ljava/lang/Class;   ClassLoader.java
ent       567 .....java.lang.BootClassLoader.getInstance ()Ljava/lang/BootClassLoader;  ClassLoader.java
xit       576 .....java.lang.BootClassLoader.getInstance ()Ljava/lang/BootClassLoader;  ClassLoader.java
xit       681 ....java.lang.ClassLoader.findLoadedClass (Ljava/lang/String;)Ljava/lang/Class;   ClassLoader.java
ent       689 ....com.uc.base.aerie.hack.ClassLoaderSupport$a.loadClass (Ljava/lang/String;Z)Ljava/lang/Class;  ProGuard
ent       704 .....java.lang.ClassLoader.getParent ()Ljava/lang/ClassLoader;    ClassLoader.java
16804 ent       726 ......java.lang.BootClassLoader.loadClass (Ljava/lang/String;Z)Ljava/lang/Class;    ClassLoader.java
ent       730 .......java.lang.ClassLoader.findLoadedClass (Ljava/lang/String;)Ljava/lang/Class;    ClassLoader.java
ent       734 ........java.lang.BootClassLoader.getInstance ()Ljava/lang/BootClassLoader;   ClassLoader.java
xit       740 ........java.lang.BootClassLoader.getInstance ()Ljava/lang/BootClassLoader;   ClassLoader.java
xit       754 .......java.lang.ClassLoader.findLoadedClass (Ljava/lang/String;)Ljava/lang/Class;    ClassLoader.java
xit       759 ......java.lang.BootClassLoader.loadClass (Ljava/lang/String;Z)Ljava/lang/Class;  ClassLoader.java
xit       763 .....java.lang.ClassLoader.loadClass (Ljava/lang/String;)Ljava/lang/Class;    ClassLoader.java
~~~

可以看到文件格式比较统一，其实正则很好去解析，但这里为了方便说明，用`pyparsing`来实现一个解析器。 将文件进行分解，假设需要解析的是`Trace` (`threadID action usecs class.method signature`)之后的数据
- 普通分割标识
    - 分号  
    `semiFlag = Literal(";")`
    - 忽略标识  
    `dotFlag = Suppress(Literal("."))`

- `threadID`标识

~~~bash
threadID =Word(nums, max=5)
~~~

-`action`标识

~~~bash
actionField = Word(alphas)
~~~

- `usecs`标识

~~~bash
usecsField = Word(nums, max=8)
~~~

- `class.method`标识

~~~bash
clsField = Word(alphas+".")
methodField = Combine("(" + ZeroOrMore(Word(alphas + ";/")) + ")" + Word(alphas + "/") + semiFlag)
~~~

- `signature`标识  
可以看到跟`clsField`一致

所以整合起来，脚本可以很快写出来（这里没有适配部分异常情况，直接`try..except`处理简单看看效果）

~~~python
import os
from pyparsing import Word, nums, Combine, alphas, Literal, ZeroOrMore, Group, \
    Suppress

semiFlag = Literal(";")
dotFlag = Suppress(Literal("."))
multiDot = ZeroOrMore(dotFlag)
threadID =Word(nums, max=5)
actionField = Word(alphas)
usecsField = Word(nums, max=8)
clsField = Word(alphas+".")
methodField = Combine("(" + ZeroOrMore(Word(alphas + ";/")) + ")" + Word(alphas + "/") + semiFlag)

regex = threadID + actionField + usecsField + multiDot + Group(clsField + methodField) + clsField

with open(os.path.join(os.getcwd(), "StepBeforeFirstDraw_o.txt"), "rb") as f:
    lineno = 0
    flag = 0
    while 1:
        line = f.readline()
        lineno += 1
        if "threadID action usecs" in line:
            flag = lineno
            continue
        if flag > 0:
            try:
                regex.parseString(line).toXML("")
            except Exception as e:
                pass
~~~

解析后的结果为：

~~~bash
/usr/bin/python2.7 /home/alex/workspace/virtual_space/project/calclex.py
['16804', 'ent', '528', ['java.lang.ClassLoader.loadClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '543', ['java.lang.ClassLoader.loadClass', '(Ljava/lang/String;Z)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '548', ['java.lang.ClassLoader.findLoadedClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '567', ['java.lang.BootClassLoader.getInstance', '()Ljava/lang/BootClassLoader;'], 'ClassLoader.java']
['16804', 'xit', '576', ['java.lang.BootClassLoader.getInstance', '()Ljava/lang/BootClassLoader;'], 'ClassLoader.java']
['16804', 'xit', '681', ['java.lang.ClassLoader.findLoadedClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '704', ['java.lang.ClassLoader.getParent', '()Ljava/lang/ClassLoader;'], 'ClassLoader.java']
['16804', 'ent', '726', ['java.lang.BootClassLoader.loadClass', '(Ljava/lang/String;Z)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '730', ['java.lang.ClassLoader.findLoadedClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '734', ['java.lang.BootClassLoader.getInstance', '()Ljava/lang/BootClassLoader;'], 'ClassLoader.java']
['16804', 'xit', '740', ['java.lang.BootClassLoader.getInstance', '()Ljava/lang/BootClassLoader;'], 'ClassLoader.java']
['16804', 'xit', '754', ['java.lang.ClassLoader.findLoadedClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'xit', '759', ['java.lang.BootClassLoader.loadClass', '(Ljava/lang/String;Z)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'xit', '763', ['java.lang.ClassLoader.loadClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'xit', '771', ['java.lang.ClassLoader.loadClass', '(Ljava/lang/String;Z)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'xit', '774', ['java.lang.ClassLoader.loadClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '809', ['java.lang.ClassLoader.loadClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '814', ['java.lang.ClassLoader.loadClass', '(Ljava/lang/String;Z)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '818', ['java.lang.ClassLoader.findLoadedClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '822', ['java.lang.BootClassLoader.getInstance', '()Ljava/lang/BootClassLoader;'], 'ClassLoader.java']
['16804', 'xit', '827', ['java.lang.BootClassLoader.getInstance', '()Ljava/lang/BootClassLoader;'], 'ClassLoader.java']
['16804', 'xit', '842', ['java.lang.ClassLoader.findLoadedClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '853', ['java.lang.ClassLoader.getParent', '()Ljava/lang/ClassLoader;'], 'ClassLoader.java']
['16804', 'xit', '857', ['java.lang.ClassLoader.getParent', '()Ljava/lang/ClassLoader;'], 'ClassLoader.java']
['16804', 'ent', '861', ['java.lang.ClassLoader.loadClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '865', ['java.lang.BootClassLoader.loadClass', '(Ljava/lang/String;Z)Ljava/lang/Class;'], 'ClassLoader.java']
['16804', 'ent', '869', ['java.lang.ClassLoader.findLoadedClass', '(Ljava/lang/String;)Ljava/lang/Class;'], 'ClassLoader.java']
~~~