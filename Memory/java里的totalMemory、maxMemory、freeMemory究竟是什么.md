# java里的totalMemory()、maxMemory()、freeMemory()究竟是什么

来源:[www.jcodecraeer.com](http://www.jcodecraeer.com/a/chengxusheji/java/2014/0917/1687.html)

* totalMemory() ：返回 Java 虚拟机中的内存总量。
* maxMemory() ：返回 Java 虚拟机试图使用的最大内存量。
* freeMemory()  ：返回 Java 虚拟机中的空闲内存量。

这是API的解释。

我写了这么一段代码

```
public class RuntimeDemo{
	public static void main(String[] args) throws Exception{
		//构造方法被私有化了。
		Runtime myRun = Runtime.getRuntime();
		System.out.println("已用内存" + myRun.totalMemory());
		System.out.println("最大内存" + myRun.maxMemory());
		System.out.println("可用内存" + myRun.freeMemory());
		String i = "";
		long start = System.currentTimeMillis();
		System.out.println("浪费内存中.....");
		for(int j = 0;j < 10000;j++){
			i += j;
		}
		long end = System.currentTimeMillis();
		System.out.println("执行此程序总共花费了" + ( end - start )+ "毫秒");
		System.out.println("已用内存" + myRun.totalMemory());
		System.out.println("最大内存" + myRun.maxMemory());
		System.out.println("可用内存" + myRun.freeMemory());
		myRun.gc();
		System.out.println("清理垃圾后");
		System.out.println("已用内存" + myRun.totalMemory());
		System.out.println("最大内存" + myRun.maxMemory());
		System.out.println("可用内存" + myRun.freeMemory());

	}
}
```

下面是打印的结果

```
已用内存31064064
最大内存460455936
可用内存30714320
浪费内存中.....
执行此程序总共花费了659毫秒
已用内存193593344
最大内存460455936
可用内存150298760
清理垃圾后
已用内存193724416
最大内存460455936
可用内存192511928
```

关于可用内存的解释：

> 我们知道，JAVA程序本身是不能直接在计算机上运行的，它需要依赖于硬件基础之上的操作系统和JVM（JAVA虚拟机）。JAVA程序启动时JVM都会分配一个初始内存和最大内存给这个应用程序。这个初始内存和最大内存在一定程度上会影响应用程序的性能。
> 
> JVM其实就是操作系统上的一个普通程序（进程名叫java，这个程序可以解释执行class文件，系统中当前运行了多少个java程序就会有多少个java进程）。当java进程启动时会首先分配一块堆内存（最小内存），以后每当java程序要求JVM（java进程）分配内存时，JVM就会在预先分配的那块内存上为java程序分配内存，当预先分配的那块内存用完时，JVM会再向操作系统要内存（物理内存），但是JVM不会无限制的向操作系统要内存，当它占用的实际内存达到一个预定值（最大可用内存）时，如果java程序还向JVM要内存，并且JVM无法通过垃圾回收机制回收当前堆中的内存来为java程序服务时，它就会给程序抛出异常：java.lang.OutOfMemoryError。其中内存回收时机并不是在用掉内存达到最大可用内存时才进行，它的运行时机是不确定的。可见JVM的最大可用内存就是java程序能够使用的最大内存。例如：我们把某JAVA程序的JVM最大可用内存设为200M，而我们的物理内存是1G。这种情况下，我们的java程序最多能使用200M内存，虽然我们可能还有800M的内存可用，但是当我们的程序用掉200M后，如果再要内存，JVM不会因为我们还有800M的内存而为我们分配内存，它会抛出java.lang.OutOfMemoryError异常。