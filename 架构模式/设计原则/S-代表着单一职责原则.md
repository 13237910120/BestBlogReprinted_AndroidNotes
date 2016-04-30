# S 代表着单一职责原则

来源:[Realm](https://realm.io/cn/news/donn-felker-solid-part-1/)

这是五个系列文章的第一部分，关于 [SOLID 原则](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design\))。

SOLID 是面向对象设计五原则的缩写：

* 单一职责原则（本文）
* 开-闭原则
* 里氏替换原则
* 接口隔离原则
* 依赖倒置原则
* *迪米特法则(Law of Demeter,Least Knowledge Principle 最少知识原则 )*

过去几周，我深入谈及到了每个原则，解释它们的含义，它们和安卓开发的关系。在这个系列的结尾，你会牢固地掌握这些核心原则的含义，理解为什么它们对安卓开发如此重要和你该如何在每天的安卓开发中使用它们。

## SOLID 背景

SOLID 最初始于 2000 年左右，由 [Robert Martin (AKA: Uncle Bob)](https://twitter.com/unclebobmartin) 和 Michael Feathers 一起提出。当这五个基础的面向对象的设计原则一起提出的时候，它帮助开发者来开发可维护和可扩展的软件。

如果你对 Uncle Bob 或 Michael Feathers 不熟悉，我强烈推荐你看看他们的一些著作。Uncle Bob’s 的著作 “[Agile Software Development, Principles, Patterns and Practices](http://amzn.to/1pzQgxn)” 和 “[Clean Code](http://amzn.to/1Uc1d5g)” 是软件社区的名著。Michael Feathers 的书 “[Working Effectively with Legacy Code](http://amzn.to/1UJcPwG)” 是我带团队的时候要求所有开发者必读的著作。它帮助你思考如何改造你的旧代码，并且使它的可维护性强起来。更重要的是，它帮你重构你对遗留代码的理解。提示 - 你的代码有测试吗？没有！？！好吧，你的代码可能已经……你认为是……遗留代码了。

阅读这些书在我的职业生涯里起了非常关键的作用，所以我建议每个开发者把这些书放在他们的读书列表里，然后把它们放在书架上以备不时地查看。

我记得我个人是在2003年的多个 .NET 项目里使用了这些 SOLID 原则。那个时候，SOLID 原则好似一阵清风，因为我的 .NET 代码在没有什么架构和指导的情况下，已经变成一团糟了。这不仅仅是 .NET 的问题，在新技术出现的时候，它常常出现（比如，安卓移动开发，还有其他）。新技术最终会达到一个成熟的水平来适用于这些 SOLID 讨论，讨论讨论它如何并且为什么这么重要。

最近，Uncle Bob 的 [Clean Architecture talk](https://vimeo.com/43612849) 在安卓开发社区里重新热起来，所以我觉得现在是时候解释一下 Uncle Bob 在他的书里面提出的一些基本原则。这个系列的文章会谈及 [SOLID](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design\)) 原则 和它们与安卓开发是如何联系起来的。

## 第一部分：单一职责原则

单一原则十分好理解。它如下描述：

<b><i><font size="5">一个类应该有且只有一个发生改变的原因。</font></i></b>

让我们看看这个例子 [RecyclerView](http://developer.android.com/reference/android/support/v7/widget/RecyclerView.html) 和它的 [adatper](http://developer.android.com/reference/android/support/v7/widget/RecyclerView.Adapter.html)。正如你所知道的那样，一个 RecyclerView 是一个灵活的视图，它可以显示一组数据到屏幕上。为了让这些数据显示到屏幕上，我们需要一个`RecyclerView adapter`。

一个 adapter 从它的数据组里获取数据，然后和视图做匹配。 一个适配器最有用的部分按理说就是`onBindViewHolder`方法（有时候是 ViewHolder 本身，但是为了简洁起见，我们仅仅关注`onBindViewHolder`）。RecyclerView 的适配器有一个原则：把一个对象和它相关的需要显示在屏幕上的视图对应起来。

假设有这些对象，`RecyclerView.Adapter`按如下实现：

```
public class LineItem {
    private String description; 
    private int quantity; 
    private long price; 
    // ... getters/setters

}

public class Order {
    private int orderNumber; 
    private List<LineItem> lineItems = new ArrayList<LineItem>();  
    // ... getters/setters

}

public class OrderRecyclerAdapter extends RecyclerView.Adapter<OrderRecyclerAdapter.ViewHolder> {
 
    private List<Order> items;
    private int itemLayout;
 
    public OrderRecyclerAdapter(List<Order> items, int itemLayout) {
        this.items = items;
        this.itemLayout = itemLayout;
    }
 
    @Override public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View v = LayoutInflater.from(parent.getContext()).inflate(itemLayout, parent, false);
        return new ViewHolder(v);
    }
 
    @Override public void onBindViewHolder(ViewHolder holder, int position) {
        // TODO: bind the view here 

    }
 
    @Override public int getItemCount() {
        return items.size();
    }
 
    public static class ViewHolder extends RecyclerView.ViewHolder {
        public TextView orderNumber;
        public TextView orderTotal;
 
        public ViewHolder(View itemView) {
            super(itemView);
            orderNumber = (TextView) itemView.findViewById(R.id.order_number);
            orderTotal = (ImageView) itemView.findViewById(R.id.order_total);
        }
    }
}
```

在上面的例子中，onBindViewHolder 是空的。我看到过的一个实现可能看起来如下：

```
@Override 
public void onBindViewHolder(ViewHolder holder, int position) {
    Order order = items.get(position);
    holder.orderNumber.setText(order.getOrderNumber().toString());
    long total = 0;
    for (LineItem item : order.getItems()) {
        total += item.getPrice();
    }
    NumberFormat formatter = NumberFormat.getCurrencyInstance(Locale.US);
    String totalValue = formatter.format(cents / 100.0); // Must divide by a double otherwise we'll lose precision

    holder.orderTotal.setText(totalValue)
    holder.itemView.setTag(order);
}
```

这段代码违反了单一职责的原则。

为什么？

适配器的 onBindViewHolder 方法不仅仅是实现 Order 对象到视图的对应关系，它还完成了价格计算和格式化的功能。这违反了单一职责原则。适配器应该仅仅负责适配一个 order 对象到它的视图表示。onBindViewHolder 执行了本不属于它的两个额外功能。

为什么这是个问题？

一个类中包含多个原则会导致许多问题。

* 第一，计算逻辑现在和适配器耦合了。如果你需要在别处显示一个 order 的总数（你很有可能需要这样做），你就会有重复的逻辑了。一旦如此，你的应用就会有一个我们都熟悉的传统软件的冗余逻辑的问题。比如，你在一个地方更新了逻辑而忘记了在另一个地方做同样的事情。你明白了。

* 第二个问题和第一个问题类似 - 你耦合了格式化逻辑和适配器。如果你需要删除或者更新呢？最终，我们会让类完成比它应该有的责任更多的工作，而且现在的应用会更容易引起缺陷，因为在一个地方有了太多的责任。

谢天谢地，这个简单的例子可以很容易的改进，只需要把 order 总数计算的逻辑抽取出来放到 Order 对象里，然后把货币格式的功能移到一个货币格式化的类中就可以了。这个格式化类也可以被 Order 使用。

一个更新后的 onBindViewHolder 方法看起来像这样：

```
@Override 
public void onBindViewHolder(ViewHolder holder, int position) {
    Order order = items.get(position);
    holder.orderNumber.setText(order.getOrderNumber().toString());
    holder.orderTotal.setText(order.getOrderTotal()); // A String, the calculation and formatting moved elsewhere

    holder.itemView.setTag(order);
}
```

我肯定你可能在想 “好吧，这很简单。这太简单了不是吗？”。它总是这么简单吗？在软件实际中大多数的答案是 - “好吧，这取决于实际情况……”。

让我们再深入一点……

## “责任”到底意味着什么？

没有谁比 Uncle Bob 说的更好了，让我直接引用如下：

<b><i><font size="5">在单一职责原则（SRP）中我们这样定义职责，“改变的原因”。如果你能想到多于一个改变类的动机，那么那个类就有多于一个的责任。</font></i></b>

实际情况是，有些时候这很难看出来 - 特别是你在代码里沉寂了很久以后。这个时候，一句名言能提醒你：

<b><i><font size="5">只见树木，不见森林。</font></i></b>

在软件上下文中，这句话意思是你太关注你的代码的细节而忘记了整体。例如 - 你工作的类看起来很棒，但是那是因为你已经很熟悉这些类了，以至于你很难看到它已经有了多个责任了。

挑战是：你什么时候运用 SRP，什么时候不用。拿刚才的适配器举例，如果你再审视一边代码，你看到各种不同的改进点，在不同的地方因为不同的原因都有必要发生改变。

```
public class OrderRecyclerAdapter extends RecyclerView.Adapter<OrderRecyclerAdapter.ViewHolder> {
 
    private List<Order> items;
    private int itemLayout;
 
    public OrderRecyclerAdapter(List<Order> items, int itemLayout) {
        this.items = items;
        this.itemLayout = itemLayout;
    }
 
    @Override public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View v = LayoutInflater.from(parent.getContext()).inflate(itemLayout, parent, false);
        return new ViewHolder(v);
    }
 
    @Override public void onBindViewHolder(ViewHolder holder, int position) {
        Order order = items.get(position);
        holder.orderNumber.setText(order.getOrderNumber().toString());
        holder.orderTotal.setText(order.getOrderTotal()); // Move the calculation and formatting elsewhere

        holder.itemView.setTag(order);
    }

 
    @Override public int getItemCount() {
        return items.size();
    }
 
    public static class ViewHolder extends RecyclerView.ViewHolder {
        public TextView orderNumber;
        public TextView orderTotal;
 
        public ViewHolder(View itemView) {
            super(itemView);
            orderNumber = (TextView) itemView.findViewById(R.id.order_number);
            orderTotal = (ImageView) itemView.findViewById(R.id.order_total);
        }
    }
}
```

适配器渲染一个视图，它把一个 order 和一个视图绑定起来，构建了一个视图持有者，等等。这个类有着多个责任。

这些责任应该被拆分开来吗？

这最终取决于你的应用是如何改变的。如果你的应用一变，你的视图组成也会发生改变，而且它的连接函数（视图的逻辑）也会发生改变。那么按照 Uncle Bob 说的，这个设计看起来比较死板，因为一个改变会导致另外一个改变。视图结构的改变也要求适配器本身发生变化，这就是设计上的死板。然而，也有人说如果你的应用不需要这种会使得多个功能在不同时刻都发生变化的改变，那么就没有必要拆分它们。在这种情况下，拆分会带来不必要的复杂性。

所以，我们该如何做？

## 一个解释僵化的例子

让我们假设一个新产品需求，当 order 的总数是零的时候要有声明，视图需要在屏幕上显示一个黄色的”免费”图像而不是数字的总数。这部分逻辑应该加到哪里？一个方法是，你需要一个 TextView，另一个方法是，你需要一个 ImageView。有两个地方你需要修改：

* 视图里
* 表示层逻辑中

我看到的一些应用中这样处理，把这些逻辑加到适配层中。不幸的是，这要求 Adapter 在视图发生变化的时候也需要修改。如果这部分逻辑在适配器里面，那这就要求适配器的逻辑如果发生改变，视图的代码也要跟着变。这就给适配器增加了另外一个责任。

这也是 [数据-视图-主导器](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter) 模式所强调的必要的解耦，这样做，类就不会变得特别死板，软件才有扩展、组合和测试的灵活性。例如，视图会实现一个接口，这个接口定义了它是如何和别人交互的，主导器会执行必要的逻辑。数据-视图-主导器模式中的主导器只负责视图、显示的逻辑，别的都不负责。

把这部分逻辑从适配器移到主导器会使得适配器更聚合，更符合单一职责原则。

这不是全部……

如果你深入看看任何 RecyclerView 适配器，你可能已经意识到适配器做了许多的事情。适配器还做了的事情是：

* 渲染视图
* 创建视图持有者
* 回收视图持有者
* 提供条目计数
* 其他。

由于 SRP 是单一职责，你可能想知道这些行为中的一部分到底该不该坚持运用 SRP 原则给提取出来。重申一遍，我将引用 Uncle Bob Martin 的话，因为他的解释实在是太准确了：

<b><i><font size="5">变化的指针只在变化真正发生时起作用。如果没有任何征兆，应用 SRP 或者其它原则都是不明智的。</font></i></b>

适配器仍然包含着各种功能，事实上，就是这样设计的。毕竟，RecyclerView 适配器是一个简单的对于 [适配器模式](https://en.wikipedia.org/wiki/Adapter_pattern) 的实现。在这个例子里，保持视图渲染和视图持有者的机制有意义；这是这个类的职责，而且是最好的实现。然而，引入额外的行为（比如视图逻辑）破坏了 SRP 原则，通过使用数据-视图-主导器模式或其他重构方式可以避免这个问题。

## 结论

单一职责原则可能是 SOLID 原则中最简单的原则了，因为它不言自明（再说一次） -

<b><i><font size="5">一个类应该有且只有一个发生改变的原因。</font></i></b>

当然，它也是个非常难以运用的原则。坚持认为需要使用 SRP 会很容易导致过度分析代码，这样你只会发现，如果你使用了 SRP，只是增加了应用的复杂性。我的建议是试着退一步，然后客观地审视代码。移除对代码的情感依赖，你将会发现你有一双清澈的眼睛。如果你这样做，你可能会从代码中发现你之前从来不曾了解的东西。你会意识到你需要使用单一职责原则，或者你可能意识到你已经做得很好了。无论如何，花点时间，多想想。

最后，如果你的应用改变了，你会发现你需要在你以前认为不需要采用 SRP 的地方使用 SRP。这完全没有问题，而且推荐这样做。

编码快乐。：）

看看这个系列的 [第二部分 英文版](https://realm.io/news/donn-felker-solid-part-2/)！