---
layout: article
title: 浴室沉思：聊聊DAL和Repository
key: 20180120blog01
tags:

    - DAL
    - Repository
    - DDD
    - 浴室沉思

lang: zh
author:       "杜小非"
---

# 这是一个由DDD群引发的随笔

---

在写了上一篇随笔[《浴室沉思：聊聊ORM》](https://www.mutuduxf.com/2018/01/05/%E6%B5%B4%E5%AE%A4%E6%B2%89%E6%80%9D-%E8%81%8A%E8%81%8AORM.html)后一些朋友私聊我，很多刚接触DDD的朋友会对Repository（仓储层）这东西有点疑惑，为什么要叫仓储层？是不是三层的DAL换个名字而已？毕竟大家都是对数据库的操作嘛，而且我用EF还有必要用仓储么？是不是可以省掉这一层？ABP框架的持久化究竟有什么问题？

在回答这些问题之前，我们要先了解一些模型分层的概念。

## DO（Domain Object）

---

几乎每位程序员在刚入门时都以三层架构（3-tier architecture）为基础，界面层（User Interface layer）、业务逻辑层（Business Logic Layer）、数据访问层（Data access layer）首次让我们在架构级别了解高内聚低耦合的概念。其中对数据访问层也即是DAL的定义是——屏蔽对“数据文件”的操作，这里的数据文件指的是数据库以及其它持久化实现。

>扩展阅读：持久化不一定是数据库，直接写个excel甚至txt也算持久化。例如rabbitmq或者redis就是通过对文件的读写来实现。追根溯源，其实数据库也是如此，例如SQL server的mdf文件。

在最简单的三层中，我们一般有一个models层或者entities层贯通三层用于数据的传递（为了便于描述后文统一叫entities层）。在界面层，entity用于显示数据或者作为view object的转换源；在业务逻辑层，和业务代码组合起来完成业务逻辑的实现；在数据访问层则作为被持久化的对象保存到数据库里。

不管怎么分，即便是后来的MVC，在很长一段时间的绝大部分项目里其实都以这种逻辑来分层，但到了DDD中就遇到了问题，因为DDD的核心模型是DO，而且DO是充血模型。

>扩展阅读：所谓充血模型，实际上就是包含了业务逻辑的对象。回想一下面向对象的定义，对象是包含“状态”（字段）以及“行为”（方法/函数）的，其实这很容易理解——贫血模型实际上就是数据的容器，它不是“活的”对象，在任何地方都可以对其进行修改，这实际上违反了高内聚的原则。打个比方：我们现在有个程序员的对象，他要减肥，那么我们要通过调用“健身”这个方法将其Weight属性逐渐降下来，而不是直接在外面set他的体重。
>贫血模型大行其道有其历史原因，最早的EJB就是将业务对象分为属性和业务方法两部分，然后spring延续了下来，但spring他爹也说：这实际上是一种面向过程的做法。在这里我们暂时不展开充血模型和贫血模型的讨论。

在DDD的模型划分中，最核心的部分就是DO（Domain Object领域对象）。DO是依据不变性划分出来的业务对象，其属性是public get protected set的，只能通过DO本身的方法来进行操作。举个简单的不严谨的例子，例如520网购，你在帮女盆友清空购物车的时候一般生成一个订单对象，其包含了子订单（例如口红、神仙水），我们大概地将DO设计如下：

```csharp
pubic class Order
{
    public string OrderId {public get; protected set;}
    public Status Status {public get; protected set;}
    public DateTime CreateTime {public get; protected set;} = Datetime.Now;
    public DateTime? PayTime {public get; protected set;}
    public DateTime? CommitTime {public get; protected set;}
    public List<OrderItem> Items {public get; protected set;}

    public void Pay(decimal money)
    {
        if(Items?.Count == 0)
            throw new exception ("请选择商品。");
        var totalPrice = Items.Sum(item => item.UnitPrice * item Quantity);
        if(money < totalPrice)
            throw new exception ("余额不足。")
        PayTime = DateTime.Now;
        Status = OrderStatus.已支付;
    }

    public void Commit()
    {
        if(Status != OrderStatus.已支付)
            throw new exception ("请先支付。");
        CommitTime = DateTime.Now;
        Status = OrderStatus.已确认;
    }
}

public class OrderItem
{
    public Guid Id {public get; protected set;}
    public string Sku {public get; protected set;}
    public decimal UnitPrice {public get; protected set;}
    public int Quantity {public get; protected set;}
}

public enum OrderStatus
{
    待支付,
    已支付,
    已确认,
    .....
}
```

在这个例子中我们可以条件反射般地想到，RDB中有两张表Order和OrderItem，这两张表是一对多的关系。假如我们用的是DAL，那么可能有个OrderDal和OrderItemDal来将数据分别持久化到RDB中，更严谨些的话还会将它们放到一个事务里执行。当然也可以有个泛型的DBHelper将Order和OrderItem分别持久化。

在这里DAL接收的参数是Order和OrderItem，它封装了对RDB的操作，让我们可以用更友好的方式来使用RDB。但这里我们要思考一个问题：所谓持久化，我们要持久化的是什么？

## PO（Persistent Object/持久化对象）

---

实际上我们持久化的并不是DO，而是DO的“状态”。例如你吃饭时被人偷拍，照片会将你吃饭时的样子记录下来，但吃饭这种“行为”本身无法持久化，如上面的Pay和Commit两个方法。另外DO的一个原则是“原子性”，也就是说拿就整个拿，存就整个存，如果是用仓储层来持久化，则只会有一个OrderRepository，将整个Order作为参数丢进去，仓储层内则将Order和OrderItem的状态部分转为OrderPo和OrderItemPo两种贫血对象，并用之持久化。

>重点：只有聚合才有仓储

到了这里我们有了一个初步的概念：DAL是对数据文件操作的封装，不管是否实现了泛型与表的转换，它都是以数据文件为中心，在使用的时候其实我们是以一种面向数据库编程的思维来进行操作；仓储层则是反过来为领域层服务，领域层需要什么它才提供什么，屏蔽掉底层持久化的具体实现。

## 我们已经用了ORM（EF）了，还有必要用仓储层么？

---

其实还是要的，上一篇随笔[《关于ORM的浴室沉思》](https://www.mutuduxf.com/2018/01/05/%E5%85%B3%E4%BA%8EORM%E7%9A%84%E6%B5%B4%E5%AE%A4%E6%B2%89%E6%80%9D.html)说明了ORM的概念，但遗憾的是现在所有ORM实际上都没有彻底解决阻抗失配的问题。

另外DO是原子性的，因此只有聚合才有仓储，上述例子OrderItem不属于聚合，它是没有仓储的。假如直接用EF的话代码是可以直接访问到DbContext\<OrderItem>，这就对代码造成了隐患。

而且直接使用EF的话，所有的业务代码将会对EF造成高度耦合，这会造成潜在的技术风险。假如我们使用的是仓储层，到时候只需要在仓储层捣鼓就行。实际上DDD项目中，ORM反而不是必要的东西。

## 所以ABP的问题是？

---

ABP是一个很好的框架，土牛的技术水平也很高，但不能说ABP就是DDD的标准答案，例如其仓储层实际上并不是服务于DO而是PO。因此有种说法ABP不是DDD而是DDD LITE，这种说法有其原因。另外仓储层严谨地说不应该提供IQueryable，而应该是Command端需要什么的获取方法才提供，否则会无法进行单元测试。

至于Query端，怎么快怎么来，管它呢~