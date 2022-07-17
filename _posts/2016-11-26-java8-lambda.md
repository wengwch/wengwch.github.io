---
title: Java8之lambda表达式
date: 2016-11-26 22:44:50
categories:
  - Programing Language
tags:
  - Java
---

Lambda表达式是Java8最重要也最令人期待的特性，它使得Java初步具有了函数式编程的能力。虽然Lambda表达式的本质只是基于接口的语法糖，但是依然可以给开发带来便利，尤其是Stream的引入，使对集合的操作变的更加方便和强大。

<!-- more -->

### lambda 表达式

lambda表达式格式: `(param1, param2, ...) -> expression`, 如果不能用一个表达式表示运行的代码，可以使用以下方式:

``` java
(param1, param2, ...) -> {
    statement1;
    statement2;
    ...
    statementN;
    return expr;
}
```

### 函数式接口

含有单一方法的接口称为SAM(Single Abstract Method，单抽象方法)接口。函数式接口是对旧有SAM接口的增强，它允许我们用lambda表达式取代传统的匿名类来实例化一个接口。要声明一个函数式接口也很容易，只需在SAM接口加上`@FunctionalInterface`注解。

```java
@FunctionalInterface
public interface MyFunction<T> {
  T apply(T t);
}
```

lambda表达式和匿名内部类类似，不过也有一些区别。

- 匿名内部类访问外部变量时，外部变量需要声明为final。lambda表达式没有这个限制，但是编译器会隐式的把变量当成final的。因此在lambda表达式中不能修改外部变量。
- 匿名内部类中的this指向该匿名类对象，而lambda中指向外部类对象，类似于闭包的概念。


### 方法引用和构造器引用

方法引用使用`::`操作符将方法名和对象或类名分隔。常用以下三种形式：

- 对象::实例方法
- 类::静态方法
- 类::实例方法

前两种情况等同于提供方法参数的lambda表达式。如`System.out::println`等同于`x -> System.out.println(x)`，`Math::pow`等同于`(x, y) -> Math.pow(x, y)`。

第三种情况，第一个参数会成为执行方法的对象。如`String::compareTo`等同于`(x, y) -> x.compareTo(y)`。

构造器引用和方法引用类似，只不过方法名是new。如`Person::new`。


### Stream

Java文档对stream包的描述如下:
> Classes to support functional-style operations on streams of elements, such as map-reduce transformations on collections
> 

stream支持对元素流进行函数式风格的操作，比如说对集合进行map-reduce的变换。stream和集合类似，但是也有一些不同的特点。

- stream 不存储值，只担当从输入源引出的管道角色，一直连接到终结操作上产生输出。
- stream 从设计上就偏向函数式风格，避免与状态发生关联。例如 filter() 操作在返回筛选结果的 stream 时，并不会改动底下的集合。
- stream的中间操作总是缓求值的
- stream 可以没有边界(无限长)。
- stream 像 Iterator 的实例一样，也是消耗品，在终结操作之后必须重新生成新的 stream 才能再次操作。

stream的操作被分为中间操作(intermediate operations)和终结操作(terminal operations)。中间操作返回一个新的stream并且总是缓求值的，例如filter并不会真的执行过滤操作，只是生成一个新的stream。终结操作遍历stream，产生结果和副作用，如forEach，collect。

中间操作又进一步被分为有状态和无状态操作。无状态操作可以独立的处理每个元素，并不依赖之前的元素，如filter，map。而有状态的操作在处理元素时，必须依赖并保存之前元素的状态，如distinct，sorted。

此外，有些操作被称为短路操作(short-circuiting operations)。如果一个中间操作是短路的，当它和一个无限输入一起使用时，会产生一个有限结果的stream，如limit。如果一个终结操作是短路的，当它和一个无限输入一起使用时，会在有限时间内完成终结操作。


### 结束

这篇仅仅是一些概念的整理和记录，并不涉及lambda表达式以及Stream的具体使用。有兴趣可以自行研究。


### 参考

[Stream operations and pipelines](http://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html#StreamOps)

[函数式编程思维](https://book.douban.com/subject/26587213/)

[写给大忙人看的Java SE 8](https://book.douban.com/subject/26274206/)









 