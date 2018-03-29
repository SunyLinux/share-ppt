title: DDD: Specification
speaker: 乌鸦
url: 
transition: slide3
files: /js/demo.js,/css/demo.css,/js/zoom.js
theme: moon
usemathjax: yes

<!--  -->


[slide]
# Specification In DDD



<!--  -->


[slide]

https://en.wikipedia.org/wiki/Specification_pattern

![](\img\spec.jpg)

<!--  -->


[slide]

<iframe style="position: absolute;
    height: 100%;
    left: -60px;
    top: -40px;
    right: 0px;
    width: 120%;" data-src="https://en.wikipedia.org/wiki/Specification_pattern" src="about:blank;"></iframe>


[slide]
## Intro

> In computer programming, the specification pattern is a particular software design pattern, whereby business rules can be recombined by chaining the business rules together using boolean logic. The pattern is frequently used in the context of domain-driven design.


<!-- 在计算机编程中，规范模式是一种特定的软件设计模式，通过使用布尔逻辑将业务规则链接在一起，可以重组业务规则。 该模式经常用于领域驱动设计的上下文中。 -->

[slide]
## Intro

> A specification pattern outlines a business rule that is combinable with other business rules. In this pattern, a unit of business logic inherits its functionality from the abstract aggregate Composite Specification class. The Composite Specification class has one function called IsSatisfiedBy that returns a boolean value. After instantiation, the specification is "chained" with other specifications, making new specifications easily maintainable, yet highly customizable business logic. Furthermore, upon instantiation the business logic may, through method invocation or inversion of control, have its state altered in order to become a delegate of other classes such as a persistence repository.

<!-- 规范模式概述了可与其他业务规则组合的业务规则。 在此模式中，业务逻辑单元从抽象聚合复合规格类继承其功能。 Composite Specification类有一个名为IsSatisfiedBy的函数，它返回一个布尔值。 实例化后，规范与其他规范“链接”，使新规范易于维护，但具有高度可定制的业务逻辑。 

配置化 ??

此外，在实例化时，业务逻辑可以通过方法调用或控制反转来改变其状态，以便成为诸如持久性存储库之类的其他类的委托。 -->

[slide]
##Scene
-----
业务规则，对象校验，集合查询，对象如何创建

> Predicate ?!

[slide]
##Code
-----
```java
class EvenSpec extends CompositeSpecification<Integer> {
    public boolean isSatisfiedBy(@NotNull Integer candidate) {
        return candidate % 2 == 0;
    }
}
class PositiveSpec extends CompositeSpecification<Integer> {
    public boolean isSatisfiedBy(@NotNull Integer candidate) {
        return candidate > 0;
    }
}

int[] queryInt(Specification<Integer> spec) {
    return IntStream.range(0, 11).filter(it -> spec.isSatisfiedBy(it)).toArray();
}

EvenSpec even = new EvenSpec();
PositiveSpec positive = new PositiveSpec();
for (int i : queryInt(even.and(positive)))  System.out.println(i); 
for (int i : queryInt(even.not().and(positive)))  System.out.println(i); 
```

<!--  -->

[slide]

## My Specification

```java
@FunctionalInterface
public interface Specification<T> {
    boolean isSatisfiedBy(@NotNull T candidate);

    default Specification<T> and(@NotNull Specification<? super T> other) {
        return candidate -> isSatisfiedBy(candidate) && other.isSatisfiedBy(candidate);
    }
    default Specification<T> andNot(@NotNull Specification<? super T> other) {
        return candidate -> isSatisfiedBy(candidate) && !other.isSatisfiedBy(candidate);
    }
    default Specification<T> or(@NotNull Specification<? super T> other) {
        return candidate -> isSatisfiedBy(candidate) || other.isSatisfiedBy(candidate);
    }
    default Specification<T> orNot(@NotNull Specification<? super T> other) {
        return candidate -> isSatisfiedBy(candidate) || !other.isSatisfiedBy(candidate);
    }
    default Specification<T> not() { 
        return candidate -> !isSatisfiedBy(candidate);
    }
}
```


[slide]
## Thinking

1. 排除 default 方法, Specification 接口仅包含 isSatisfiedBy 一个方法, 仍旧是 FunctionalInterface, 可以改写为 lambda
2. default 方法的设计目的是为了让已经存在的接口可以演化, 可以方便的对已经实现 Specification 的接口的类进行功能升级, 扩展 Specification 的组合方式
3. 给 Specification 构造加点甜度, 比如 

```java
Specification2<Integer> init = a -> true; 
init.and(num -> num % 2 == 0).and(num -> num > 0);
```

<!--  -->




[slide]
## Coding

```java
int[] queryInt(Specification<Integer> spec) {
    return IntStream.range(0, 11)
                    .filter(it -> spec.isSatisfiedBy(it)).toArray();
}

Specification<Integer> spec = num -> true;
Specification<Integer> even = num -> num % 2 == 0;
Specification<Integer> positive = num -> num > 0;

for (int i : queryInt(even.not().and(positive))) {
    System.out.println(i);
}

for (int i : queryInt(spec.and(num -> num % 2 == 0).and(num -> num > 0))) {
    System.out.println(i);
}
```



<!-- ## Query: Query Object / Query Strategies -->



[slide]
# End