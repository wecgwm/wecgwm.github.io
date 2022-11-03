---
title: Collector and Stream.reduce()
date: 2022-11-03 13:05:39
tags: Java
---

## 背景

[删除字符串中的破折号和空字符](https://leetcode.cn/problems/reformat-phone-number/)

> 当然，最简洁的方式是  
>         String s = number.replace(" ", "").replace("-", "");
> 但这里暂时忽略这种方案，只是为了指出该背景

一种较复杂的方案是使用 Stream，例如
```Java
private String justCase(String str){  
    return str.chars()  
            .mapToObj(__ -> (char) __)  
            .filter(c -> c != '-' && c != ' ')
            .collect(Collector.of(  
                    StringBuilder::new,  
                    StringBuilder::append,  
                    (__, ___) -> {throw new UnsupportedOperationException("un support parallel stream");},
                    StringBuilder::toString));  
}
```

在这里，要将字符数组重新收集成 `String` 的合理做法应该只有使用：
```Java
.collect(supplier,  accumulator,  combiner);
// or
.collect(Collector.of());  
```

其余做法例如 `mapToObj(String::valueOf).collect(joining)` 会频繁创建 String；
而 `reduce` 严格来讲既不是**可变规约**也没有简洁多少
```Java
private String justCase(String str){  
    return  str.chars().parallel()  
            .mapToObj(__ -> (char) __)  
            .filter(c -> c != '-' && c != ' ')  
            .reduce(new StringBuilder(),  
                    StringBuilder::append,  
                    StringBuilder::append).toString();  
}
```

## 实例化 CollectorImpl

> 这里只对 `Collector` 做该话题下的简单介绍，详细知识见 javadocs

Collector 接口由四个抽象函数指定，这些函数协同工作以将条目累积到可变结果容器中，并可选择对结果执行最终转换。他们是：
- supplier() ：创建一个新的结果容器
- accumulator() ：将新数据元素合并到结果容器中
- combiner()：将两个结果容器合并为一个（  ）
- finisher() ：对容器执行可选的最终转换


`Collector` 接口在 JDK 中的实现类位于 `Collectors.CollectorImpl`
而实例化  `CollectorImpl` 有两种途径：
- 通过 `Collectors` 类预定义的例如 `Collectors.toList()` 等静态工厂方法实例化
- 通过 `Collector` 接口中暴露出的两种 `Collector.of` 方法实例化：
```Java
public interface Collector<T, A, R> {
	public static<T, R> Collector<T, R, R> of
	
											(Supplier<R> supplier,  
                                          BiConsumer<R, T> accumulator,  
                                          BinaryOperator<R> combiner,  
                                          Characteristics... characteristics) 
                                          
                                          {...}
                                          
    public static<T, A, R> Collector<T, A, R> of
    
											(Supplier<A> supplier,  
                                             BiConsumer<A, T> accumulator,  
                                             BinaryOperator<A> combiner,  
                                             Function<A, R> finisher,  
                                             Characteristics... characteristics)

                                             {...}
}
```

可以看到，`Collector` 接口只提供了两种重载用于实例化 `CollectorImpl` 
并且这两种重载都必须传入 `supplier` 、`accmulator` 以及 `combiner`，前两个参数很好理解，毕竟 `CollectorImpl` 不好对此提供默认的实现
但是对于组合器  `combiner`，**由于组合器只有在执行并发规约时会使用到**，也就是说，对于上面场景下的收集器（即不考虑使用并发流的场景），提供一个 `combiner` 并没有实际意义

## 为什么必须提供 combiner

在[该问题中](https://stackoverflow.com/questions/24308146/why-is-a-combiner-needed-for-reduce-method-that-converts-type-in-java-8/24316429#24316429)，发现这种操作在 `Scala` 中被称为`foldLeft`。需要注意的是，Java 的库函数中并没有提供等效于 `foldLeft` 的实现。

> 在上面的回答中提到：
> Finally, Java doesn't provide `foldLeft` and `foldRight` operations because they imply a particular ordering of operations that is inherently sequential. This clashes with the design principle stated above of providing APIs that support sequential and parallel operation equally.
> 最后，Java 不提供`foldLeft`and`foldRight`操作，因为它们暗示了一种特定的操作顺序，这种顺序本质上是顺序的。这与上述提供同样支持顺序和并行操作的 API 的设计原则相冲突。


虽然该说法有一定说服力，但还是继续搜索了为什么 Java 没有提供 `foldLeft`，试图继续理解所提到的`设计原则`。
但是结果却找到了[JDK-8292845](https://bugs.openjdk.org/browse/JDK-8292845)和[JDK-8292845](https://bugs.openjdk.org/browse/JDK-8292845)，但在这两个增强请求中，却没有对相关`设计原则`进行讨论，而是计划会在将来对此进行实现。

也许在不久的将来，就会有一种更合理的 folding operations 可以替换上方看似不合理的实现

## reduce vs collect

先看一段代码：
```Java
public void justCase(){  
    String number = "a-b";  s
    Function<String, Stream<Character>> function = str -> str.chars()  
            .parallel()  
            .mapToObj(__ -> (char) __)  
            .filter(c -> c != '-' && c != ' ');  
  
    String s1 = function.apply(number)  
            .reduce(new StringBuilder(),  
                    StringBuilder::append,  
                    StringBuilder::append).toString(); // baba  
  
    String s2 = function.apply(number)  
            .collect(Collector.of(  
                    StringBuilder::new,  
                    StringBuilder::append,  
                    StringBuilder::append,  
                    StringBuilder::toString)); // ab
                    
    String s3 = function.apply(number)  
            .collect(  
                    StringBuilder::new,  
                    StringBuilder::append,  
                    StringBuilder::append).toString(); // ab
}
```

^f50671

发现在 `parallel stream` 下 `reduce()` 的输出并不符合我们的预期，先查看 `reduce()` 的方法注释
```Java
...
 identity值必须是组合器函数的标识。这意味着对于所有u ，
     combiner(identity , u) == u
 此外， combiner函数必须与accumulator函数兼容；对于所有u和t ，必须满足以下条件：
     combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t)
...

<U> U reduce(U identity,  BiFunction<U, ? super T, U> accumulator,  BinaryOperator<U> combiner);
```
对这里的约定进行验证：
```Java
public void justCase(){
	// combiner(identity , u) == u  
	StringBuilder identity = new StringBuilder();  
	StringBuilder u = new StringBuilder().append('b');  
	BinaryOperator<StringBuilder> combiner = StringBuilder::append;  
	
	StringBuilder apply0 = combiner.apply(identity, u);  
	log.debug(String.valueOf(apply0.toString().equals(u.toString()))); // true  
	

	// combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t)  
	Object t = 'a';  
	BiFunction<StringBuilder, Object, StringBuilder> acc = StringBuilder::append;  
	  
	StringBuilder apply1 = acc.apply(identity, t);  
	StringBuilder apply2 = combiner.apply(u, apply1);  
	  
	StringBuilder apply3 = acc.apply(u, t);  
	
	log.debug(String.valueOf(apply2.toString().equals(apply3.toString()))); // true
}
```

发现我们的用例其实是符合 `reduce()` 方法在 javadocs 中的约定的，于是继续查看相关代码

```Java
// 使用提供的标识、累积和组合函数对该流的元素执行 归约 
reduce(U identity, BiFunction<U,? super T,U> accumulator, BinaryOperator<U> combiner)

// 对此流的元素执行 可变归约 操作
collect(Supplier<R> supplier, BiConsumer<R,? super T> accumulator, BiConsumer<R,R> combiner)
```

会发现这其实是因为 `reduce()` 只是 [Reduction operations](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/stream/package-summary.html#ReductionOperations) 导致的（而 `StringBuilder` 是可变对象），在该场景下应该使用  [Mutable reduction(可变规约)](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/stream/package-summary.html#MutableReduction) ，也就是 `collect()`


实际上仔细查看代码会发现 `reduce` 和 `collect` 的累加器 `accumulator` 定义也并不一样
```Java
// Reduction operations 在累加器中返回处理结果，处理结果的类型不能是可变的
reduce(... BiFunction<U, ? super T, U> accumulator...)
// Mutable reduction(可变规约) 在累加器中不返回处理结果而是通过修改可变容器本身
collect(...BiConsumer<R, ? super T> accumulator...) 
```

## 总结

[Mutable reduction(可变规约)](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/stream/package-summary.html#MutableReduction)

[how-does-reduce-method-work-with-parallel-streams-in-java-8](https://stackoverflow.com/questions/56023452/how-does-reduce-method-work-with-parallel-streams-in-java-8)
> The problem lies in you using Stream::reduce for mutable reduction. 
> You should instead use Stream::collect

[java-8-streams-collect-vs-reduce](https://stackoverflow.com/questions/22577197/java-8-streams-collect-vs-reduce/38728166#38728166)
>The reason is simply that:
>- collect() can only work with mutable result objects.
>- reduce() is designed to work with immutable result objects.

[java-8-streams-collect-vs-reduce](https://stackoverflow.com/questions/22577197/java-8-streams-collect-vs-reduce/22577274#22577274)
> reduce是一个“折叠”操作，它将二元运算符应用于流中的每个元素，其中运算符的第一个参数是前一个应用程序的返回值，第二个参数是当前流元素。
> collect是一种聚合操作，其中创建“集合”并将每个元素“添加”到该集合中。然后将流中不同部分的集合添加到一起。