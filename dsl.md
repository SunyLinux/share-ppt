title: DDD: DSL in Java
speaker: 乌鸦
url: 
transition: slide3
files: /js/demo.js,/css/demo.css,/js/zoom.js
theme: moon
usemathjax: yes

<!--  -->

[slide]
# DSL in Java
https://en.wikipedia.org/wiki/Domain-specific_language
<!--  -->


[slide]
Almost every type of DSL can be mapped to Java. 
-----
Let’s have a look at how this can be done.
<!--  -->




[slide]
## Rule

Domain Specific Languages are usually built up from rules that roughly look like these

```
1. SINGLE-WORD
2. PARAMETERISED-WORD parameter
3. WORD1 [ OPTIONAL-WORD ]
4. WORD2 { WORD-CHOICE-A | WORD-CHOICE-B }
5. WORD3 [ , WORD3 ... ]
```

<!--  -->

[slide]
## Grammar

```
Grammar  ::= 'SINGLE-WORD'
           | 'PARAMETERISED-WORD' '(' [A-Z]+ ')'
           | 'WORD1' 'OPTIONAL-WORD'?
           | 'WORD2' ( 'WORD-CHOICE-A' | 'WORD-CHOICE-B' )
           | 'WORD3'+
```

<!--  -->


[slide]
## Railroad Diagrams

![](\img\dls_eg1.jpg)


<!--  -->


[slide]
## Java Impl


1. 每个 DSL **关键字** 转换为一个 Java **方法**
2. 每个 DSL **连接** 转换为一个 Java **接口**
3. 当遇到**强制**选择时, 每个选择的**关键字**转换为当前**接口**的**方法**(如果只有一个关键字则只有一个方法)
4. 当遇到**可选关键字**时候，将当前**接口**继承于包含所有关键字方法的**接口**
5. 当**关键字重复**时，表示可重复**关键字**的**方法**返回**接口**自身，而不是下一个接口
6. 将DLS**子定义**作为参数时则允许**递归**

<!-- 
- Every DSL “keyword” becomes a Java method
- Every DSL “connection” becomes an interface
- When you have a “mandatory” choice (you can’t skip the next keyword), every keyword of that choice is a method in the current interface. If only one keyword is possible, then there is only one method
- When you have an “optional” keyword, the current interface extends the next one (with all its keywords / methods)
- When you have a “repetition” of keywords, the method representing the repeatable keyword returns the interface itself, instead of the next interface
- Every DSL subdefinition becomes a parameter. This will allow for recursiveness -->


<!-- Note, it is possible to model the above DSL with classes instead of interfaces, as well. But as soon as you want to reuse similar keywords, multiple inheritance of methods may come in very handy and you might just be better off with interfaces. -->

<!-- 也可以用 class 代替接口，但因为接口可以多继承，方便重用关键字/方法。 -->
<!-- 通过以上规则，可以创建任意复杂性的 DSL。你必须实现接口，不过是另一个回事了。 -->


[slide]

```java
interface Start {
    End singleWord();
    End parameterisedWord(String parameter);
    Intermediate1 word1();
    Intermediate2 word2();
    Intermediate3 word3();
}
 
interface End {
    void end();
}
 
interface Intermediate1 extends End {
    End optionalWord();
}

interface Intermediate2 {
    End wordChoiceA();
    End wordChoiceB();
}
 
interface Intermediate3 extends End {
    Intermediate3 word3();
}
```

<!-- 
Start
Initial interface, entry point of the DSL
Depending on your DSL's nature, this can also be a class with static
methods which can be static imported making your DSL even more fluent

End
Terminating interface, might also contain methods like execute();


Intermediate1 -> optional
Intermediate DSL "step" extending the interface that is returned
by optionalWord(), to make that method "optional"


Intermediate2 -> choice
Intermediate DSL "step" providing several choices (similar to Start)


Intermediate3 -> repetition
Intermediate interface returning itself on word3(), in order to allow
for repetitions. Repetitions can be ended any time because this interface extends End

 -->

<!--  -->

[slide]

```java
Start start = // ...
 
start.singleWord().end();
start.parameterisedWord("abc").end();
 
start.word1().end();
start.word1().optionalWord().end();
 
start.word2().wordChoiceA().end();
start.word2().wordChoiceB().end();
 
start.word3().end();
start.word3().word3().end();
start.word3().word3().word3().end();
```



[slide]
# Practice - SQL

https://dev.mysql.com/doc/refman/5.7/en/select.html

```SQL
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
    select_expr [, select_expr ...]
    [FROM table_references
    [WHERE where_condition]
    [GROUP BY {col_name | expr | position}
      [ASC | DESC], ... ]
    [HAVING where_condition]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC], ...]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]
    [FOR UPDATE]]
```


[slide]
## Railroad Diagrams - select-core
![select-core](https://www.sqlite.org/images/syntax/select-stmt.gif)
https://www.sqlite.org/syntaxdiagrams.html#select-stmt


[slide]
## Railroad Diagrams - select-core
![select-core](https://www.sqlite.org/images/syntax/select-core.gif)
https://www.sqlite.org/syntaxdiagrams.html#select-stmt

<!--  -->

[slide]
## Railroad Diagrams - compound-select-stmt
![compound-select-stmt](https://www.sqlite.org/images/syntax/compound-select-stmt.gif)
https://www.sqlite.org/syntax/compound-select-stmt.html


[slide]

## Coding...



[slide]
## Thinking

顺序连接在一起的关键字可以简化为一个方法; select all/distict 可简化处理, e.g. group by , order by

<!--  -->



[slide]
## Thinking

观察铁路图, 到 where 的连接有两条，一条 select -> where, 另一条 from -> where
---
select -> where 可以通过 select [ -> from ] -> where，

<!--  -->


[slide]
## Thinking

clause: select ... [from ...] [group by ...] [order by ...] [limit ...] 除了 select 任意关键字都是可选的,  隐含了一个骨架是关键字有序的（暂时不考虑 composed bound op (union) 多条语句的情况，这是第6条原则的应用）
    1. select;
    2. select from;
    3. select groupby;
    4. select orderby;
    5. select limit;
    6. select from ordery limit
    7. .....

<!--  -->


[slide]
## Thinking - Extending
继承意味着拥有父类的方法, 因为第一个原则, 方法为关键字, 所以继承意味着可选的跳过当前类的关键字, 实现 optional 的能力;
所以, 按顺序把可选关键字的连接声明完成之后, 就会发现 select 的实现拥有了直达其他子句关键字(方法)的能力

<!--  -->


[slide]
## Thinking - Returning
按照第二个原则, 连接即接口,这里按照连接终点的关键字来命名接口名称,连接终点的关键字则成为该接口的方法, 而连接真正起作用的点在关键字/方法的返回值上, 按照 clause 的顺序, select 关键字 后是 from 关键字, 所以 Select 接口的 select 方法返回 From 接口, 即可以调用 From 接口的 from 关键字, 或者跳过 from , 直接调用其继承而来的方法/关键字;
<!--  -->




<!-- 

```
interface DSL {}
interface From extends Where {}
interface Where extends Group {}
interface Group extends Order {}
interface Order extends Limit {}
interface Limit extends End {}
interface End {}
```

先名名称 End, 将 Union 时候再重命名成 Select

```
interface Select {
    From select(String ...fields);
    From selectAll();
    From selectDistinct(String ...fields);
}
```


order by asc/desc 是一个单独的子句, 不能采用以上继承的关系, 会构造出 select.select().from("").where().asc() 这种非法结构;
asc/desc 本身是可选的, 需要使用继承, asc/desc 是二选一, 声明两个方法即可;
因为orderby 是单独子句, order 要直接继承 Limit;

```
interface Order extends Limit {
    AscDesc orderBy(String filed);
}

interface AscDesc extends Limit {
    Limit asc();
    Limit desc();
}
    
select.select().from("").where().orderBy("").asc().limit(0);
select.select().from("").where().orderBy("").limit(0);
```

```
interface Order extends AscDesc {
    AscDesc orderBy(String filed);
}

interface AscDesc extends Limit {
    Limit asc();
    Limit desc();
}
```    

limit offset 的处理：
limit(row_count)
limit(offset, row_count)
offset().limit() 这种情况的处理方式跟 order by asc/desc 处理方式相同



然后 讲 where 条件的处理

```
interface Field {
    Condition eq(String a);
    Condition like(String a);
    BetweenAnd between(String a);
    // ...
}
    
interface BetweenAnd {
    Condition and(String b);
}
    
interface Condition {
    Condition and(Condition c);
    Condition or(Condition c);
    Condition not();
        
        // 后面补充一下 这种实现
    default Condition andNot(Condition c) {
        return and(c).not();
    }

    default Condition orNot(Condition c) {
        return or(c).not();
    }
}
    
Field f = null;
Condition c = f.between("").and("").not().and(
    f.eq("").and(f.like("xxx"))
);
```

补充 where 的参数 Condition


然后 写一个简单的实现

TODO::



最后补充 其他语法也可以类似的转换，比如 ES，只要遵循以上6个原则，但是需要小心处理下类型；
其中 构建查询的 DSL 比较有用；
DDD 中有个 CQRS~ 查询与命令模式，其中查询可以简单构建 DSL 来实现，或者 specification，或者 queryObject 实现, 暴露一个 query 接口
而 其他 增删改等操作，可以采用 Command Pattern 来做；
好处可以统一领域层接口，query  结构会产生很多 Command 的实现逻辑；
每个 service 都实现都包含一个 commandExecutor, 各种 API 的 impl 最后表面一个门面, 内部全部实现为
CommandExecutor.execute (xxxCommand)
下回讲解；....


 -->




<!-- ## Query: Query Object / Query Strategies -->
 


[slide]
# End
----

... Rule Engine with Groovy DSL