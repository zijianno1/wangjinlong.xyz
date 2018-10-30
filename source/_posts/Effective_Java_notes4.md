---
title: 《Effective Java》学习笔记四——泛型
categories:
    - '技术'
    - 'Java'
tags:
    - Java
    - 读书笔记
---

《Effective Java》学习笔记
<!--more-->



# 请不要在新代码中使用原生态类型

声明中具有一个或者多个类型参数的类或者接口，就是泛型类或者接口。泛型类和接口统称为泛型。

每种泛型定义一组参数化的类型。每个泛型都定义一个原生态类型，即不带任何实际类型参数的泛型名称。

如果使用原生态类型，就失掉了泛型在安全性和表述性方面的所有优势。如果使用像List这样的原生态类型，就会失掉类型安全性，但是如果使用像List<Object\>这样的参数化类型，则不会。

无限制的通配符类型。如果要使用泛型，但是不确定或者不关心实际的类型参数，就可以使用一个问号代替。通配符类型是安全的，原生态类型则不安全。由于可以将任何元素放进使用原生态类型的集合中，因此很容易破坏该集合的类型约束条件。

不要在新代码中使用原生态类型，有两个小例外，两者都源于“泛型信息可以再运行时被擦除”这一事实。在类文字中必须使用原生态类型，规范不允许使用参数化类型。换句话说，List.class，String[].class和int.class都合法，但是List<String\>.class和List<?\>.class则不合法。第二个例外与instanceof操作符有关。由于泛型信息可以在运行时被擦除，因此在参数化类型而非无限制通配符类型上使用instanceof操作符是非法的。

总之，使用原生态类型会在运行时导致异常，因此不要在新代码中使用。原生态类型只是为了与引入泛型之前的遗留代码进行兼容性和互用而提供的。

快速回顾：Set<Obejct\>是个参数化类型，表示可以包含任何对象类型的一个集合；Set<?\>则是一个通配符类型，表示只能包含某种未知对象类型的一个集合；Set则是个原生态类型，它脱离了泛型系统。前两种是安全的，最后一种不安全。

| 术语             | 示例                                |
| ---------------- | ----------------------------------- |
| 参数化的类型     | List<String\>                      |
| 实际类型参数     | String                              |
| 泛型             | List<E\>                           |
| 形式类型参数     | E                                   |
| 无限制通配符类型 | List<?\>                           |
| 原生态类型       | List                                |
| 有限制类型参数   | <E extends Number\>                |
| 递归类型限制     | <T extends Comparable<T\>\>       |
| 有限制通配符类型 | List<?extends Number\>             |
| 泛型方法         | static <E\>List<E\> asList(E[] a) |
| 类型令牌         | String.class                        |



# 消除非受检警告

用泛型编程时，会遇到许多编译器警告：非受检强制转化警告、非受检方法调用警告、非受检普通数据创建警告，以及非受检转换警告。

要尽可能地消除每一个非受检警告。

如果无法消除警告，同时可以证明引起警告的代码是类型安全的，（只有在这种情况下才）可以用一个@SuppressWarnings("unchecked")注解来禁止这条警告。

SuppressWarnings注解可以用在任何粒度的级别中，从单独的局部变量声明到整个都可以。应该始终在尽可能小的范围中使用SuppressWarnings注解。永远不要在整个类上使用SuppressWarnings。

每当使用SuppressWarnings("unchecked")注解时，都要添加一条注释，说明为什么这么做是安全的。



# 列表优先于数组

数组与泛型相比，有两个重要的不同点。首先，数组是协变的，即，如果Sub是Super的子类型，那么数组类型Sub[]就是Super[]的子类型。相反，泛型则是不可变的：对于任意两个不同的类型Type1和Type2，List<Type1\>既不是List<Type2\>的子类型，也不是List<Type2\>的超类型。这实际上意味着数组是有缺陷的。

利用数组，为数组元素赋值时，若元素类型不匹配，则要在运行时才能发现所犯的错误；但是利用列表，则可以在编译时发现错误。

数组与泛型之间的第二大区别在于，数据是具体化的。因此数组会在运行时才知道并检查它们的元素类型约束。

创建泛型数组是非法的。因为它不是类型安全的。

当你得到泛型数组创建错误时，最好的解决办法通常是优先使用集合类型List<E\>，而不是数组类型E[]。这样可能会损失一些性能或者简洁性，但是换回的却是更高的类型安全性和互用性。



# 优先考虑泛型

使用泛型比使用需要在客户端代码中进行转换的类型来得更加安全，也更加容易。在设计新类型的时候，要确保它们不需要这种转换就可以使用。这通常意味着要把类做成是泛型的。只要时间允许，就把现有的类型都泛型化。这对于这些类型的新用户来说会变得更加轻松，又不会破坏现有的客户端。



# 优先考虑泛型方法

静态工具方法尤其适合于泛型化。Collections中的所有“算法”方法都泛型化了。

泛型方法的一个显著特征是，无需明确指定类型参数的值，不像调用泛型构造器的时候是必须指定的。编译器通过检查方法参数的类型来计算类型参数的值。

相关的模式是泛型单例工厂。有时，会需要创建不可变但又适合于许多不同类型的对象。由于泛型是通过擦除实现的，可以给所有必要的类型参数使用单个对象，但是需要编写一个静态工厂方法，重复地给每个必要的类型参数分发对象。这种模式最常用于函数对象，如Collections.reverseOrder，但也适用于像Collections.emptySet这样的集合。



# 利用有限制通配符来提升API的灵活性

为了获得最大限度的灵活性，要在表示生产者或者消费者的输入参数上使用通配符类型。

下面的助记符便于让你记住要使用那种通配符类型：

***PECS表示producer-extends，consumer-super。***

如果参数化类型表示一个T生产者，就使用<? extends T\>；如果它表示一个T消费者，就使用<? super T\>。

如果使用得当，通配符类型对于类的用户来说几乎是无形的。他们使方法能够接受它们应该接受的参数，并拒绝那些应该拒绝的参数。如果类的用户必须考虑通配符类型，类的API或许就会出错。



# 优先考虑类型安全的异构容器

有时会需要未限定固定数目的类型参数的容器，此时，可以将容器的键进行参数化而不是将容器参数化。然后将参数化的键交给容器来插入或者获得值。用泛型系统来确保值的类型和它的键相符。