---
layout: post
title: "[每周一译]在Java中提升函数以更好地“函数式”编程"
date: 2017-08-18 19:29:34 +0800
comments: true
categories: java java8 translate
---

Java8中的Stream和Optional给我们带来了函数式编程的乐趣，但Java仍然缺少很多函数编程的关键特性。Lambda表达式、Optional和Stream只是函数式编程的冰山一角。这也导致了[varvr](https://github.com/vavr-io/vavr)和[functionlajava](https://github.com/functionaljava/functionaljava)这些类库的出现，他们都源于Haskell这个纯函数式编程语言。

如果想要更加地“函数式”编程，那么首先要注意的是不要过早的中断monad(一种设计模式，表示将一个运算过程通过函数拆解成互相连接的多个步骤。只要提供下一步运算所需的函数，整个运算就会自动进行下去, Optional、Stream都是monad)，比如，很多人经常会在还不需要的时候就调用了Optional.get()和Stream.collect()提前终止monad。本文主要讲述如何通过提升方法来使得代码更"函数式"。

<!--more-->

假设有一个接口可以对数字进行计算。

```
public interface Math {
    int multiply(int a, int b);
    double divide(int a, int b);
    ..
}
```

我们要使用这个接口来对使用Optional做包装的数字做计算。

```
public interface NumberProvider {
    Optional<Integer> getNumber();
}
```

接着我们来实现一个方法能够返回两个数字相除的结果，结果用Optional包装。如果这两个数字有一个为空则返回空Optional。如下：

```
public Optional<Double> divideFirstTwo(NumberProvider numberProvider, Math math) {
    Optional<Integer> first = numberProvider.getNumber();
    Optional<Integer> second = numberProvider.getNumber();
    if(first.isPresent() && second.isPresent()) {
        double result = math.divide(first.get(), second.get());
        return Optional.of(result);
    } else {
        return Optional.empty();
    }
}
```

上面的代码非常不优雅，有大量的代码都是在做Optional的包装和解包装。可以让上面的代码变得更加“函数式”，如下：

```
public Optional<Double> divideFirstTwo(NumberProvider numberProvider, Math math) {
    return numberProvider.getNumber()
           .flatMap(first -> numberProvider.getNumber()
                                     .map(second -> math.divide(first, second)));
}
```

这样代码少了很多，也优雅了很多。先调用第一个Optional的flatMap，再在lambda中调用第二个Optional的map，进一步可以抽取出一个提升方法：

```
public interface Optionals {
    static <R, T, Z> BiFunction<Optional<T>, Optional<R>, Optional<Z>> lift(BiFunction<? super T, ? super R, ? extends Z> function) {
        return (left, right) -> left.flatMap(leftVal -> right.map(rightVal -> function.apply(leftVal, rightVal)));
    }
}
```

如上，可知这个方法提升能够提升任何具有两个Optional参数、一个Optional结果的函数，使得被提升的函数具有Optional的一个特性：如果一个参数是空的，那么结果就是空的。如果JDK抽取flatMap和map到一个公共接口，如Monad，那么我们可以为Java Monad的每一个实例(Stream、Optional、自己的实现类)实现一个公共的提升函数。但现实是我们不得不为每一个实例都复制粘贴上面的代码。最终的divideFirstTwo代码如下：

```
import static com.ps.functional.monad.optional.Optionals.lift;
...
public Optional<Double> divideFirstTwo(NumberProvider numberProvider, Math math) {
    return lift(math::divide).apply(numberProvider.getNumber(), numberProvider.getNumber());
}
```

**ps: 此文内容来自<https://dzone.com/articles/lifting-functions-to-work-with-monads-in-java?edition=311409&edition=0&utm_source=Zone%20Newsletter&utm_medium=email&utm_campaign=java%202017-08-01>，加入了本人的理解和认知。**