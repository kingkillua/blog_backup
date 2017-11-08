---
title: spring事务传播行为
date: 2016-01-01 10:44:18
tags:
categories: 学
---

## 大概
什么是事务就不多说了
```java
Public interface PlatformTransactionManager()...{  
    // 由TransactionDefinition得到TransactionStatus对象
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException; 
    // 提交
    Void commit(TransactionStatus status) throws TransactionException;  
    // 回滚
    Void rollback(TransactionStatus status) throws TransactionException;  
    } 
```
然后spring只是定义了事务TransactionDefinition,给了这些接口，具体怎么实现怎么去回滚就交给数据源自己弄了
获得事务对象，提交，回滚，加起来就这3件事，但真实情况会有多个事务相互调用，还有嵌套之类的情况
所以为了处理好这些情况，TransactionDefinition里面定义了事务的这些属性：
1传播行为，2隔离级别，3回滚规则，4事务超时，5是否只读
主要理解传播行为这点

<!--more-->
## 传播行为
复制粘贴一波～
事务的传播行为是指，如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。
在TransactionDefinition定义中包括了如下几个表示传播行为的常量：
* **PROPAGATION_REQUIRED** 	当前方法必须运行在事务中。如果当前事务存在，方法将会在该事务中运行。否则，会启动一个新的事务
* **PROPAGATION_SUPPORTS** 	当前方法不必须在事务中，但是如果存在当前事务的话，那么该方法会在这个事务中运行
* **PROPAGATION_MANDATORY** 	当前方法必须在事务中运行，如果当前事务不存在，则会抛出一个异常
* **PROPAGATION_REQUIRED_NEW** 	当前方法必须运行在它自己的事务中。一个新的事务将被启动。如果已经存在当前事务，
   在该方法执行期间，当前事务会被挂起。如果使用JTATransactionManager的话，则需要访问TransactionManager
* **PROPAGATION_NOT_SUPPORTED** 	当前方法不应该运行在事务中。如果存在当前事务，在该方法运行期间，当前事务将被挂起。
   如果使用JTATransactionManager的话，则需要访问TransactionManager
* **PROPAGATION_NEVER** 	当前方法不应该运行在事务上下文中。如果当前正有一个事务在运行，则会抛出异常
* **PROPAGATION_NESTED**       当前已经存在一个事务，那么当前方法将会在嵌套事务中运行。嵌套的事务可以独立于当前事务进行单独地提交或回滚。
  如果当前事务不存在，那么其行为与PROPAGATION_REQUIRED一样。注意各厂商对这种传播行为的支持是有所差异的。可以参考资源管理器的文档来确认它们是否支持嵌套事务

#### 拿事务嵌套举个栗子：
Service A 的 Method A() 调用 内层Service B 的 Method B()
```java
void methodA() {
   ServiceB.methodB();
}
```
* 1.Method B()是 PROPAGATION_REQUIRED
假如执行Method A(),spring 起了个事务，这时执行Method B()，直接就都在一个事务里了，如果失败一起回滚
如果执行Method B()时没有在事务中，那它自己另外起一个事务，这个时候失败回滚就只是B自己的事了
* 2.Method B()是 PROPAGATION_REQUIRES_NEW
Method B()必须运行在自己的事务中，如果Method A()已经起了个事务了，A你先暂停挂着，B先执行
跟上面不一样的是，A，B不同一个事务，B如果提交了，就算A失败了，也只能A自己滚，滚不了B
B如果失败了，A不一定回滚，得看B抛出的异常是不是A会回滚的异常
* 3.Method B()是 PROPAGATION_SUPPORTS
B看到A起了事务就加进去，没有事务，那它也不起事务。。。
* 4.Method B()是 PROPAGATION_NESTED
Method A()如果没起事务，B就是默认的PROPAGATION_REQUIRED，起一个自己的事务，只管自己
Method A()如果起了个事务，那执行B作为嵌套子事务运行在A中

#### REQUIRES_NEW和NESTED区别
主要事2，4的区别，很多人搞不明白，甚至还有根据英文原文理解错的
引用3段话
<blockquote>PROPAGATION_REQUIRES_NEW starts a new, independent "inner" transaction for the given scope. 
This transaction will be committed or rolled back completely independent from the outer transaction, 
having its own isolation scope, its own set of locks, etc. The outer transaction will get suspended 
at the beginning of the inner one, and resumed once the inner one has completed. ... </blockquote>
  
<blockquote>PROPAGATION_NESTED on the other hand starts a "nested" transaction, which is a true 
subtransaction of the existing one. What will happen is that a savepoint will be taken at the start
of the nested transaction. Íf the nested transaction fails, we will roll back to that savepoint.
The nested transaction is part of of the outer transaction, so it will only be committed at the end 
of the outer transaction. </blockquote>
  
<blockquote>Rolling back the entire transaction is the choice of the demarcation code/config that started the 
outer transaction.So if an inner transaction throws an exception and is supposed to be rolled back 
(according to the rollback rules), the transaction will get rolled back to the savepoint taken at 
the start of the inner transaction. The immediate calling code can then decide to catch the exception
and proceed down some other path within the outer transaction.
If the code that called the inner transaction lets the exception propagate up the call chain, the 
exception will eventually reach the demarcation code of the outer transaction. At that point, the 
rollback rules of the outer transaction decide whether to trigger a rollback. That would be a 
rollback of the entire outer transaction then.
So essentially, it depends on your exception handling. If you catch the exception thrown by the 
inner transaction, you can proceed down some other path within the outer transaction. If you let 
the exception propagate up the call chain, it's eventually gonna cause a rollback of the entire 
outer transaction.</blockquote>

#### 总结
嵌套事务是外部事务的子事务，提交必须得等外部事务一起提交，回滚则会回滚到savepoint点
从使用上去理解就是，你可以一个事务里面多个嵌套事务，相当于设置多个存档点，
如果都捕获嵌套事务的异常，那大家各跑各的，成功的先等着，失败的回到执行前，然后都等最后一起提交
如果不捕获嵌套事务异常，那嵌套事务异常直接触发外部回滚，那大家一起回滚，这种就很坑了不推荐




