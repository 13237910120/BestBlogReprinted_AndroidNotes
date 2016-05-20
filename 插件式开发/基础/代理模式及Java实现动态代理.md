# 代理模式及Java实现动态代理

来源:[http://www.jianshu.com/p/6f6bb2f0ece9](http://www.jianshu.com/p/6f6bb2f0ece9)

## 代理模式

定义：给某个对象提供一个代理对象，并由代理对象控制对于原对象的访问，即客户不直接操控原对象，而是通过代理对象间接地操控原对象。

![](proxy-pattern/1.gif)

在上图中：

* RealSubject 是原对象（本文把原对象称为"委托对象"），Proxy 是代理对象。
* Subject 是委托对象和代理对象都共同实现的接口。
* Request() 是委托对象和代理对象共同拥有的方法。

要理解代理模式很简单，其实生活当中就存在代理模式：

> 我们购买火车票可以去火车站买，但是也可以去火车票代售处买，此处的火车票代售处就是火车站购票的代理，即我们在代售点发出买票请求，代售点会把请求发给火车站，火车站把购买成功响应发给代售点，代售点再告诉你。
但是代售点只能买票，不能退票，而火车站能买票也能退票，因此代理对象支持的操作可能和委托对象的操作有所不同。

再举一个写程序会碰到的一个例子：

> 如果现在有一个已有项目（你没有源代码，只能调用它）能够调用 int compute(String exp1) 实现对于后缀表达式的计算，你想使用这个项目实现对于中缀表达式的计算，那么你可以写一个代理类，并且其中也定义一个compute(String exp2)，这个exp2参数是中缀表达式，因此你需要在调用已有项目的 compute() 之前将中缀表达式转换成后缀表达式(Preprocess)，再调用已有项目的compute()，当然你还可以接收到返回值之后再做些其他操作比如存入文件(Postprocess)，这个过程就是使用了代理模式。

在平时用电脑也会碰到代理模式的应用：

> 远程代理：我们在国内因为GFW，所以不能访问 facebook，我们可以用翻墙（设置代理）的方法访问。访问过程是：<br/>
> (1)用户把HTTP请求发给代理<br/>
> (2)代理把HTTP请求发给web服务器<br/>
> (3)web服务器把HTTP响应发给代理<br/>
> (4)代理把HTTP响应发回给用户<br/>

Java 实现上面的UML图的代码（即实现静态代理）为：

```
public class ProxyDemo {
    public static void main(String args[]){
        RealSubject subject = new RealSubject();
        Proxy p = new Proxy(subject);
        p.request();
    }
}

interface Subject{
    void request();
}

class RealSubject implements Subject{
    public void request(){
        System.out.println("request");
    }
}

class Proxy implements Subject{
    private Subject subject;
    public Proxy(Subject subject){
        this.subject = subject;
    }
    public void request(){
        System.out.println("PreProcess");
        subject.request();
        System.out.println("PostProcess");
    }
}
```

代理的实现分为：

* **静态代理：**代理类是在编译时就实现好的。也就是说 Java 编译完成后代理类是一个实际的 class 文件。
* **动态代理：**代理类是在运行时生成的。也就是说 Java 编译完之后并没有实际的 class 文件，而是在运行时动态生成的类字节码，并加载到JVM中。

## Java 实现动态代理

在前一节我们已经介绍了静态代理（第一节已经实现）和动态代理，那么在 Java 中是如何实现动态代理，即如何做到在运行时动态的生成代理类呢？

首先先说明几个词：

* 委托类和委托对象：委托类是一个类，委托对象是委托类的实例。
* 代理类和代理对象：代理类是一个类，代理对象是代理类的实例。

Java实现动态代理的大致步骤如下：

* 1、定义一个委托类和公共接口。
* 2、自己定义一个类（调用处理器类，即实现 InvocationHandler 接口），这个类的目的是指定运行时将生成的代理类需要完成的具体任务（包括Preprocess和Postprocess），即代理类调用任何方法都会经过这个调用处理器类（在本文最后一节对此进行解释）。
* 3、生成代理对象（当然也会生成代理类），需要为他指定(1)委托对象(2)实现的一系列接口(3)调用处理器类的实例。因此可以看出一个代理对象对应一个委托对象，对应一个调用处理器实例。

Java 实现动态代理主要涉及以下几个类：

* java.lang.reflect.Proxy: 这是生成代理类的主类，通过 Proxy 类生成的代理类都继承了 Proxy 类，即 DynamicProxyClass extends Proxy。
* java.lang.reflect.InvocationHandler: 这里称他为"调用处理器"，他是一个接口，我们动态生成的代理类需要完成的具体内容需要自己定义一个类，而这个类必须实现 InvocationHandler 接口。
Proxy 类主要方法为：

```
//创建代理对象  
static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)
```

这个静态函数的第一个参数是类加载器对象（即哪个类加载器来加载这个代理类到 JVM 的方法区），第二个参数是接口（表明你这个代理类需要实现哪些接口），第三个参数是调用处理器类实例（指定代理类中具体要干什么）。这个函数是 JDK 为了程序员方便创建代理对象而封装的一个函数，因此你调用newProxyInstance()时直接创建了代理对象（略去了创建代理类的代码）。其实他主要完成了以下几个工作：

```
static Object newProxyInstance(ClassLoader loader,
	Class<?>[] interfaces,InvocationHandler handler){
    //1. 根据类加载器和接口创建代理类
    Class clazz = Proxy.getProxyClass(loader, interfaces); 
    //2. 获得代理类的带参数的构造函数
    Constructor constructor = clazz.getConstructor(new Class[] { InvocationHandler.class });
    //3. 创建代理对象，并制定调用处理器实例为参数传入
    Interface Proxy = (Interface)constructor.newInstance(new Object[] {handler});
}
```

Proxy 类还有一些静态方法，比如：

* InvocationHandler getInvocationHandler(Object proxy): 获得代理对象对应的调用处理器对象。
* Class getProxyClass(ClassLoader loader, Class[] interfaces): 根据类加载器和实现的接口获得代理类。

Proxy 类中有一个映射表，映射关系为：(<ClassLoader>,(<Interfaces>,<ProxyClass>) )，可以看出一级key为类加载器，根据这个一级key获得二级映射表，二级key为接口数组，因此可以看出：一个类加载器对象和一个接口数组确定了一个代理类。

我们写一个简单的例子来阐述 Java 实现动态代理的整个过程：

```
public class DynamicProxyDemo01 {
    public static void main(String[] args) {
        RealSubject realSubject = new RealSubject();    //1.创建委托对象
        ProxyHandler handler = new ProxyHandler(realSubject);    //2.创建调用处理器对象
        Subject proxySubject = (Subject)Proxy.newProxyInstance(RealSubject.class.getClassLoader(),
                                                        RealSubject.class.getInterfaces(), handler);    //3.动态生成代理对象
        proxySubject.request();    //4.通过代理对象调用方法
    }
}

/**
 * 接口
 */
interface Subject{
    void request();
}

/**
 * 委托类
 */
class RealSubject implements Subject{
    public void request(){
        System.out.println("====RealSubject Request====");
    }
}
/**
 * 代理类的调用处理器
 */
class ProxyHandler implements InvocationHandler{
    private Subject subject;
    public ProxyHandler(Subject subject){
        this.subject = subject;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        System.out.println("====before====");//定义预处理的工作，当然你也可以根据 method 的不同进行不同的预处理工作
        Object result = method.invoke(subject, args);
        System.out.println("====after====");
        return result;
    }
}
```

InvocationHandler 接口中有方法：

```
invoke(Object proxy, Method method, Object[] args)
```

这个函数是在代理对象调用任何一个方法时都会调用的，方法不同会导致第二个参数method不同，第一个参数是代理对象（表示哪个代理对象调用了method方法），第二个参数是 Method 对象（表示哪个方法被调用了），第三个参数是指定调用方法的参数。

动态生成的代理类具有几个特点：

* 继承 Proxy 类，并实现了在Proxy.newProxyInstance()中提供的接口数组。
* public final。
* 命名方式为 $ProxyN，其中N会慢慢增加，一开始是 $Proxy1，接下来是$Proxy2...
* 有一个参数为 InvocationHandler 的构造函数。这个从 Proxy.newProxyInstance() 函数内部的clazz.getConstructor(new Class[] { InvocationHandler.class }) 可以看出。

Java 实现动态代理的缺点：因为 Java 的单继承特性（每个代理类都继承了 Proxy 类），只能针对接口创建代理类，不能针对类创建代理类。

> 不难发现，代理类的实现是有很多共性的（重复代码），动态代理的好处在于避免了这些重复代码，只需要关注操作。

## Java 动态代理的内部实现

现在我们就会有一个问题： Java 是怎么保证代理对象调用的任何方法都会调用 InvocationHandler 的 invoke() 方法的？

这就涉及到动态代理的内部实现。假设有一个接口 Subject，且里面有 int request(int i) 方法，则生成的代理类大致如下：

```
public final class $Proxy1 extends Proxy implements Subject{
    private InvocationHandler h;
    private $Proxy1(){}
    public $Proxy1(InvocationHandler h){
        this.h = h;
    }
    public int request(int i){
        Method method = Subject.class.getMethod("request", new Class[]{int.class});    //创建method对象
        return (Integer)h.invoke(this, method, new Object[]{new Integer(i)}); //调用了invoke方法
    }
}
```

通过上面的方法就成功调用了 invoke() 方法。

## 元编程

到最后，我还想介绍一下元编程，我是从 《[MacTalk·人生元编程](http://product.china-pub.com/3769393)》 中了解到元编程这个词的：

> 元编程就是能够操作代码的代码

Java 支持元编程因为反射、动态代理、内省等特性。

## 参考文献
[Java 动态代理机制分析及扩展，第 1 部分](http://www.ibm.com/developerworks/cn/java/j-lo-proxy1/)<br/>
[动态代理的作用是什么？](http://zhidao.baidu.com/link?url=lPrrnjezrVWQEAwnK3pLWAISllIeqmFBDe-H4PiSzxEczb8EZR0p6asCdsXxSfDLmRf50yoIo6q3QqpoZzoMMa)<br/>
[JDK动态代理实现原理](http://rejoy.iteye.com/blog/1627405)<br/>
[彻底理解JAVA动态代理](http://www.cnblogs.com/flyoung2008/archive/2013/08/11/3251148.html)<br/>
[慕课网：模式的秘密---代理模式](http://www.imooc.com/learn/214)