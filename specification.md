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
https://en.wikipedia.org/wiki/Specification_pattern

<!-- 
小而美

实现领域驱动设计书里头一个印象深刻的简单范式

背景，解决了什么问题，由上一次分享的争论引出  
 
-->

[slide]

![](\img\spec.jpg)

<!-- 
实现了ISpecification的对象则意味着是一个Specification，它可以通过与其他Specification对象的And，Or或对自身的取反来生成新的逻辑。为了方便这些“组合逻辑”的开发，我们还会准备一个抽象的CompositeSpecification类：

CompositeSpecification提供了构建复合Specification的基础逻辑，它提供了And、Or和Not方法的实现，让其他Specification类只需要专注于IsSatisfiedBy方法的实现即可。
-->


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


<!-- 
Specification模式的作用是构建可以自由组装的业务逻辑元素。Specification类有一个IsSatisifiedBy函数，用于校验某个对象是否满足该Specification所表示的条件。多个Specification对象可以组装起来，并生成新Specification对象，这便可以形成高度可定制的业务逻辑。

例如，我们可以使用依赖注入（控制反转）的方式来配置这个业务逻辑，以此保证系统的灵活性。
-->


[slide]
## Advantage

> When using the specification pattern we move business rules in separate specification classes. These specification classes can be easily combined by using composite specifications. In general, specification improve reusability and maintainability. Additionally specifications can easily be unit tested. 




[slide]
##Scene
-----
业务规则，对象校验，集合查询，如何对象创建

<!-- Predicate ?! -->

[slide]

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

## 
```java
/**
 * @author xiaofeng 
 */
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


<!--
Specification模式的关键在于，Specification类有一个IsSatisifiedBy函数，用于校验某个对象是否满足该Specification所表示的条件。多个Specification对象可以组装起来，并生成新Specification对象，这便可以形成高度可定制的业务逻辑。从中可以看出，一个Specification对象的关键，其实就是一个IsSatisifiedBy方法的逻辑。每种对象，一段逻辑。每个对象的唯一关键，也就是这么一段逻辑。因此，我们完全可以构造这么一个“通用”的类型，允许外界将这段逻辑通过构造函数“注入”到Specification对象中

```java
public class Specification<T> : ISpecification<T>
{
    private Func<T, bool> m_isSatisfiedBy;

    public Specification(Func<T, bool> isSatisfiedBy)
    {
        this.m_isSatisfiedBy = isSatisfiedBy;
    }

    public bool IsSatisfiedBy(T candidate)
    {
        return this.m_isSatisfiedBy(candidate);
    }
}
```

嗯嗯，这也是一种依赖注入。在普通的面向对象语言中，承载一段逻辑的最小单元只能是“类”，只是我们说，某某类中的某某方法就是我们需要的逻辑。而在C#中，从最早开始就有“委托”这个东西可用来承载一段逻辑。与其为每种情况定义一个特定的Specification类，让那个Spcification类去访问外部资源（即建立依赖），不如我们将这个类中唯一需要的逻辑给准备好，各种依赖直接通过委托由编译器自动保留，然后直接注入到一个“通用”的类中。很关键的是，这样在编程方面也非常容易。

至于原本ISpecification<T>中的And，Or，Not方法，我们可以将它们提取成扩展方法。有朋友说，既然有了扩展方法，那么对于一些不需要访问私有成员/状态的方法，都应该提取到实体的外部，避免“污染”实体。不过我不同意，在我看来，到底是用实例方法还是扩展方法，还是个根据职责和概念而一定的。我在这里打算使用扩展的目的，是因为And，Or，Not并非是一个Specification对象的逻辑，并不是一个Specification对象说，“我要去And另一个”，“我要去Or另一个”，或者“我要造……取反”。就好比二元运算符&&、||、或者+、-，左右两边的运算数字有主次之分吗？没有，它们是并列的。因此，我选择使用额外的扩展方法，而不是将这些职责交给某个Specification对象：

http://blog.zhaojie.me/2009/09/specification-pattern-in-csharp-practice.html
http://blog.zhaojie.me/2009/09/specification-pattern-in-csharp-practice-answer-1.html
http://blog.zhaojie.me/2009/09/specification-pattern-in-csharp-practice-answer-2.html

-->

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

<!--  -->
[slide]
## Query Object VS Specification

> A specific question that is related to your domain

https://stackoverflow.com/questions/16800976/differences-between-query-object-and-specification-patterns

<!--  -->

[slide]
## Query Object VS Specification
### Specification

> Evans' Specification pattern implements a business rule as an object representing a predicate on another object, an entity or value object. 

-----

A Specification can be used for several things: 

> validating an object, querying a collection or specifying how a new object is to be created.

<!--  -->



[slide] 
## Query Object VS Specification
### Specification
-----
> Fowler's Query Object pattern is a specialization of the Interpreter pattern that allows database queries to be presented in domain language. An example mostly from Fowler:

```java
QueryObject query = new QueryObject(Person.class);
query.addCriteria(Criteria.greaterThan("numberOfDependents", 0))
List<Person> persons = query.execute(unitOfWork);
```
<!--  -->


[slide]
## Query Object VS Specification

-----

> Specification is essentially the same as the Criteria class that is part of the Query Object pattern. 

-----

The Query Object description didn't intend Criteria to have any other purpose than specifying queries, but if you used both patterns in the same program you'd certainly want to use your Specifications as your Query Objects' Criteria.




<!--  -->
[slide]
## Paper

https://www.martinfowler.com/apsupp/spec.pdf

[slide]
# End