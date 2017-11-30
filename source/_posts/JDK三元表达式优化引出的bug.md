---
title: JDK三元表达式优化引出的bug
date: 2017-11-30 21:46:40
tags:
---
## 问题
今天生产版本迭代，邻组有一个NPE bug，迟迟找不到原因，导致下班推迟。直接上代码（包引用已经忽略）。
## 代码
```
public class Main {
    private Integer test;

    public Main(String cardImage, Integer test) {
        System.out.println(StringUtils.isNotEmpty(cardImage));
        this.test = StringUtils.isNotEmpty(cardImage) ? 0 : test;
    }

    public static void main(String[] args) {
        Main main = new Main(null, null);
        System.out.println(main.test);
    }
}
```
根据日志显示，我们找到了异常抛出的代码。代码初看起来没什么问题，所以我们的焦点聚集在逻辑问题上，修改了可能的几个原因，但是问题依旧存在。
所以我们将报错代码提出单独做测试。
可能是类似经验不足，或者可能程序员过分相信编译器，或者产生问题更容易产生对自己的怀疑，直到我们排除自身的问题，我们才想到所谓的Java并不是所见即所得，而当我们打开编译过后的class文件，错误才找到源头（惭愧啊！）。
我们来看编译过后的文件代码（包引用已经忽略）。
## 编译过后的代码
```
public class StringsTest {
    private Integer test;
    public StringsTest(String cardImage, Integer test) {
        System.out.println(StringUtils.isNotEmpty(cardImage));
        this.test = StringUtils.isNotEmpty(cardImage) ? 0 : test.intValue();
    }

    public static void main(String[] args) {
        StringsTest main = new StringsTest((String)null, (Integer)null);
        System.out.println(main.test);
    }
}
```
问题很显然了，
![JDk优化](/image/jdk_优化.png)

## 原因
以上代码第5行，此处编译器将三元表达式的两个结果值优化成基本类型并取值！问题显然出于此。那么为什么会这样呢？

经查阅之，在jdk1.5之前，对于三元表达式：\[条件语句\] ? \[表达式1\] : \[表达式2\],
表达式1和表达式2的类型要求必须一致。而在jdk1.5以后，由于有了自动拆箱和装箱的原因，两者只要其中一种或者两者都能被拆箱即可。比如表达式1为Integer，而表达式2为int类型的。
而我们的代码符合这个条件，看起来没有问题。

但是， 有个需要注意的是，如果表达式1和表达式2的类型不相同，那么他们需要**对交集类型的自动参考转换**。

对于封装类型和基础类型，显然编译器会将其优化成基础类型！ 
到这里，解决问题就很简单了。在外围添加条件语句，绕过这个三元表达式即可。

## 总结

1. 不要在包装类上调用构造方法。
2. 查找问题，有条件的话尽量查看源码，那个才是最接近真实的状态。
3. 相信自己。
4. 继续学习。


