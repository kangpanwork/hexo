---
title: Spring循环依赖
date: 2022-05-15 11:17:04
tags: Spring
---



1. 什么是循环依赖，怎么判断是循环依赖？
2. 有哪些循环依赖？
3. 哪些可以解决，哪些不可以解决？
4. 怎么解决？

带着这几个问题，先想一想。



以下是个人理解的答案。

1. 什么是循环依赖？

   回答这个问题之前，先简单介绍下 Bean 的生命周期。

   ![6717ac3791eb6adb2c803bf036dec58d.png](https://img.gejiba.com/images/6717ac3791eb6adb2c803bf036dec58d.png)

   

如果对象与对象在属性填充的时候互相依赖，称之为循环依赖。那么 Spring 是怎么判断对象之间是循环依赖的呢？

这里举个例子，

```Java
@Service
public class AService {

    @Autowired
    private BService bService;

    public void fun() {
    }
}

@Service
public class BService {

    @Autowired
    private AService aService;

    public void fun() {
    }
}
```

AService 在实例化的时候会用一个 Set 集合标识该 Bean 是在创建中，Set 的 key 就是 Bean Name。当进行 BService 属性填充的时候，如果 BService 的属性（AService）存在这个 Set 集合中，说明它们之间存在依赖关系。



2. 有哪些循环依赖？

   - 普通的循环依赖，参考上面代码

   - 多例的循环依赖

     ```java
     @Service
     @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
     public class AService {
     
         @Autowired
         private BService bService;
     
         public void fun() {
         }
     }
     
     @Service
     @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
     public class BService {
     
         @Autowired
         private AService aService;
     
         public void fun() {
         }
     }
     ```

     

   - 构造的循环依赖

     ```
     @Service
     public class AService {
     
         @Autowired
         private AService(BService bSerivce){...}
     
         public void fun() {
         }
     }
     
     @Service
     
     public class BService {
     
         @Autowired
         private BService(AService aSerivce){...}
     
         public void fun() {
         }
     }
     ```

   - 代理对象的循环依赖， @Async 注解 @Transactional 注解会产生代理对象，还有一些自己写的 AOP，对原始对象进行了增强。

     ```java
     @Service
     public class AService {
     
         @Autowired
         private BService bService;
     
         public void fun() {
         }
     }
     
     @Service
     public class BService {
     
         @Autowired
         private AService aService;
     
     	@Async
         public void fun() {
         }
     }
     ```

   - 指定加载顺序的循环依赖，AService 加上 @DependsOn 注解，指定 BSerivce 先加载，而 BService 又指定 AService 先加载。
   
     ```java
     @Service
     @DependsOn(value = "BService")
     public class AService {
     
         @Autowired
         private BService bService;
     
         public void fun() {
         }
     }
     
     @Service
     @DependsOn(value = "AService")
     public class BService {
     
         @Autowired
         private AService aService;
     
         public void fun() {
         }
     }
     ```

3. 哪些可以解决？哪些不可以解决？

   普通循环依赖、代理循环依赖及多例循环依赖、指定加载顺序的依赖可以解决。而构造循环依赖无法解决，因为两个对象都是有参构造，构造前都必须先实例化，存在构造依赖都无法先提前实例化（除非其中一个有无参构造）。

4. 怎么解决？

   - 普通对象依赖和代理对象依赖 ，Spring 框架使用三级缓存解决。

   ![7ae02888247955b4e552a7761fd246b4.png](https://img.gejiba.com/images/7ae02888247955b4e552a7761fd246b4.png)

   AService 

   1. 实例化对象：当获取 AService Bean 的时候先去一级缓存中查找，没有则实例化该对象，然后将获取 Bean 对象的 Lambda 方法存储到三级缓存中（代理对象依赖情况，属性填充的时候根据原始对象获取代理对象，提前进行 AOP），使用 Lambda 表达式是延迟处理，后期需要使用的时候才去处理。

   2. 填充属性：填充 BService 属性。

      BService

      2.1 先去一级缓存中查找，没有则实例化该对象，同 AService 第一步。

      2.2 填充属性：填充 AService 属性，先去二级缓存中查找，没有则从三级缓存创建对象，再把创建的对象放入二级缓存，删除三级缓存（Lambda 只需要执行一次，保持单例）。

      2.3 其它属性填充

      2.4 初始化操作

      2.5 放入一级缓存

      2.6 把 BService 从 Set 集合（标识正常创建中的 ）中移除。

   3. 其它属性填充
   4. 初始化操作
   5. 放入一级缓存
   6. 把 AService 从 Set 集合（标识正常创建中的 ）中移除。

   

   二级缓存是为了解决循环依赖的问题，也是为了满足对象单例。而三级缓存是为了解决代理循环依赖问题，虽然可以在对象实例化后进行代理，但是破坏了 Spring 的设计原则，将 AOP 设计在初始化阶段进行。而代理依赖需要在属性填充的时候进行 AOP，所以在实例化后创建了三级缓存，方便提前创建代理对象。

   

   虽然二级缓存是半成品，例如 BService 填充完 AService 的时候，AService 还没有执行其它属性填充等操作。但是当 AService 执行  3 和 4 步的时候，BService 里面的 AService 已经是成品了（对象引用）。

   

   当然有些代理循环依赖需要代码层修改，例如 @Async 注释的代理类需要在类上加上 @Lazy 注解解决，否则报错。

   

   - 多例循环依赖的修改单例可以解决，@DependsOn 注解的打破它不循环依赖可以解决。

   
