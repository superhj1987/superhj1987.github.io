---
layout: post
title: "[每周一译]更好地使用Java8中的方法引用"
date: 2017-09-02 19:29:34 +0800
comments: true
categories: java java8
---

在Java8中，使用方法引用非常简单，如String::isEmpty，但无法使用它否定的方法引用。本文内容即如何解决此问题使得我们能够更加全面地使用方法引用。

<!--more-->

首先看一个使用方法引用的例子：

```
Stream.of("A", "", "B").filter(String::isEmpty).count()
```

上面代码的输出为1，即空字符串的数目。如果我们想要获取非空字符串的数目，就不能直接使用方法引用了。

```
Stream.of("A", "", "B").filter(s -> !s.isEmpty()).count()
```

Java8中的Predicate，有predicate.negate()可以转换为断言的否定形式，但String::isEmpty却无法这么做(String::isEmpty.negate()或者!String::isEmpty)。因为方法引用并不是一个lambda或者函数接口，它能够被解析为一个或者多个函数接口。如，String::isEmpty至少可以被解析如下：

- `Predicate<String>`
- `Function<String, Boolean>`

为了解决上述的问题，我们可以通过某种机制显式地将方法引用转换为一个函数接口：

```
public static <T> Predicate<T> as(Predicate<T> predicate) {
    return predicate;
}
```

通过使用一个静态方法，接受方法引用参数，返回一个函数接口，即可实现方法引用到函数接口的转换。接着，我们就可以使用方法引用来实现上面例子中的获取非空字符串的数目。

```
Stream.of("A", "", "B").filter(as(String::isEmpty).negate()).count();
```

进一步还能使用各种组合的Predicate。

```
.filter(as(String::isEmpty).negate().and("A"::equals))
```

由于一个方法引用可能会被解析为多种函数接口，因此如果我们实现很多参数不同的as方法，那么很容易造成混淆。更好的方式则是在方法名中加入函数参数的类型来区分。

```
import java.util.function.*;

public class FunctionCastUtil {
    public static <T, U> BiConsumer<T, U> asBiConsumer(BiConsumer<T, U> biConsumer) {
        return biConsumer;
    }
    public static <T, U, R> BiFunction<T, U, R> asBiFunction(BiFunction<T, U, R> biFunction) {
        return biFunction;
    }
    public static <T> BinaryOperator<T> asBinaryOperator(BinaryOperator<T> binaryOperator) {
        return binaryOperator;
    }
    public static <T, U> BiPredicate<T, U> asBiPredicate(BiPredicate<T, U> biPredicate) {
        return biPredicate;
    }
    public static BooleanSupplier asBooleanSupplier(BooleanSupplier booleanSupplier) {
        return booleanSupplier;
    }
    public static <T> Consumer<T> asConsumer(Consumer<T> consumer) {
        return consumer;
    }
    public static DoubleBinaryOperator asDoubleBinaryOperator(DoubleBinaryOperator doubleBinaryOperator) {
        return doubleBinaryOperator;
    }
    public static DoubleConsumer asDoubleConsumer(DoubleConsumer doubleConsumer) {
        return doubleConsumer;
    }
    public static <R> DoubleFunction<R> asDoubleFunction(DoubleFunction<R> doubleFunction) {
        return doubleFunction;
    }
    public static DoublePredicate asDoublePredicate(DoublePredicate doublePredicate) {
        return doublePredicate;
    }
    public static DoubleToIntFunction asDoubleToIntFunction(DoubleToIntFunction doubleToIntFunctiontem) {
        return doubleToIntFunctiontem;
    }
    public static DoubleToLongFunction asDoubleToLongFunction(DoubleToLongFunction doubleToLongFunction) {
        return doubleToLongFunction;
    }
    public static DoubleUnaryOperator asDoubleUnaryOperator(DoubleUnaryOperator doubleUnaryOperator) {
        return doubleUnaryOperator;
    }
    public static <T, R> Function<T, R> asFunction(Function<T, R> function) {
        return function;
    }
    public static IntBinaryOperator asIntBinaryOperator(IntBinaryOperator intBinaryOperator) {
        return intBinaryOperator;
    }
    public static IntConsumer asIntConsumer(IntConsumer intConsumer) {
        return intConsumer;
    }
    public static <R> IntFunction<R> asIntFunction(IntFunction<R> intFunction) {
        return intFunction;
    }
    public static IntPredicate asIntPredicate(IntPredicate intPredicate) {
        return intPredicate;
    }
    public static IntSupplier asIntSupplier(IntSupplier intSupplier) {
        return intSupplier;
    }
    public static IntToDoubleFunction asIntToDoubleFunction(IntToDoubleFunction intToDoubleFunction) {
        return intToDoubleFunction;
    }
    public static IntToLongFunction asIntToLongFunction(IntToLongFunction intToLongFunction) {
        return intToLongFunction;
    }
    public static IntUnaryOperator asIntUnaryOperator(IntUnaryOperator intUnaryOperator) {
        return intUnaryOperator;
    }
    public static LongBinaryOperator asLongBinaryOperator(LongBinaryOperator longBinaryOperator) {
        return longBinaryOperator;
    }
    public static LongConsumer asLongConsumer(LongConsumer longConsumer) {
        return longConsumer;
    }
    public static <R> LongFunction<R> asLongFunction(LongFunction<R> longFunction) {
        return longFunction;
    }
    public static LongPredicate asLongPredicate(LongPredicate longPredicate) {
        return longPredicate;
    }
    public static <T> LongSupplier asLongSupplier(LongSupplier longSupplier) {
        return longSupplier;
    }
    public static LongToDoubleFunction asLongToDoubleFunction(LongToDoubleFunction longToDoubleFunction) {
        return longToDoubleFunction;
    }
    public static LongToIntFunction asLongToIntFunction(LongToIntFunction longToIntFunction) {
        return longToIntFunction;
    }
    public static LongUnaryOperator asLongUnaryOperator(LongUnaryOperator longUnaryOperator) {
        return longUnaryOperator;
    }
    public static <T> ObjDoubleConsumer<T> asObjDoubleConsumer(ObjDoubleConsumer<T> objDoubleConsumer) {
        return objDoubleConsumer;
    }
    public static <T> ObjIntConsumer<T> asObjIntConsumer(ObjIntConsumer<T> objIntConsumer) {
        return objIntConsumer;
    }
    public static <T> ObjLongConsumer<T> asObjLongConsumer(ObjLongConsumer<T> objLongConsumer) {
        return objLongConsumer;
    }
    public static <T> Predicate<T> asPredicate(Predicate<T> predicate) {
        return predicate;
    }
    public static <T> Supplier<T> asSupplier(Supplier<T> supplier) {
        return supplier;
    }
    public static <T, U> ToDoubleBiFunction<T, U> asToDoubleBiFunction(ToDoubleBiFunction<T, U> toDoubleBiFunction) {
        return toDoubleBiFunction;
    }
    public static <T> ToDoubleFunction<T> asToDoubleFunction(ToDoubleFunction<T> toDoubleFunction) {
        return toDoubleFunction;
    }
    public static <T, U> ToIntBiFunction<T, U> asToIntBiFunction(ToIntBiFunction<T, U> toIntBiFunction) {
        return toIntBiFunction;
    }
    public static <T> ToIntFunction<T> asToIntFunction(ToIntFunction<T> ioIntFunction) {
        return ioIntFunction;
    }
    public static <T, U> ToLongBiFunction<T, U> asToLongBiFunction(ToLongBiFunction<T, U> toLongBiFunction) {
        return toLongBiFunction;
    }
    public static <T> ToLongFunction<T> asToLongFunction(ToLongFunction<T> toLongFunction) {
        return toLongFunction;
    }
    public static <T> UnaryOperator<T> asUnaryOperator(UnaryOperator<T> unaryOperator) {
        return unaryOperator;
    }
    private FunctionCastUtil() {
    }
}

Stream.of("A", "", "B").filter(asPredicate(String::isEmpty).negate()).count();
```

英文原文：<https://dzone.com/articles/put-your-java-8-method-references-to-work>