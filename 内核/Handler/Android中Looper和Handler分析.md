# Android Looper和Handler分析

来源:[http://blog.csdn.net/innost/article/details/6055793](http://blog.csdn.net/innost/article/details/6055793)

第一次接触Android应用程序（这里指的是JAVA层的UI程序，也难怪了，Google放出的API就只支持JAVA应用程序了），很难搞明白内部是如何实现的。但是，从原理上分析，应该是有一个消息循环，一个消息队列，然后主线程不断得从消息队列中取得消息并处理之。

然而，google封装得太厉害了，所以一时半会还是搞不清楚到底是怎么做的。本文将分析android内的looper，这个是用来封装消息循环和消息队列的一个类，handler其实可以看做是一个工具类，用来向消息队列中插入消息的。好比是Windows API的SendMessage中的HANDLE，这个handle是窗口句柄。

```
//Looper类分析
//没找到合适的分析代码的办法，只能这么来了。每个重要行的上面都会加上注释
//功能方面的代码会在代码前加上一段分析
public class Looper {
   //static变量，判断是否打印调试信息。
    private static final boolean DEBUG = false;
    private static final boolean localLOGV = DEBUG ? Config.LOGD : Config.LOGV;

    // sThreadLocal.get() will return null unless you've called prepare().
//线程本地存储功能的封装，TLS，thread local storage,什么意思呢？
// 因为存储要么在栈上，例如函数内定义的内部变量。要么在堆上，例如new或者malloc出来的东西
// 但是现在的系统比如Linux和windows都提供了线程本地存储空间，
// 也就是这个存储空间是和线程相关的，一个线程内有一个内部存储空间，
// 这样的话我把线程相关的东西就存储到
//这个线程的TLS中，就不用放在堆上而进行同步操作了。
    private static final ThreadLocal sThreadLocal = new ThreadLocal();
//消息队列，MessageQueue，看名字就知道是个queue..
    final MessageQueue mQueue;
    volatile boolean mRun;
//和本looper相关的那个线程，初始化为null
    Thread mThread;
    private Printer mLogging = null;
//static变量，代表一个UI Process（也可能是service吧，这里默认就是UI）的主线程
    private static Looper mMainLooper = null;
    
     /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
//往TLS中设上这个Looper对象的，如果这个线程已经设过了ｌｏｏｐｅｒ的话就会报错
//这说明，一个线程只能设一个looper
    public static final void prepare() {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException(
            		"Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper());
    }
    
    /** Initialize the current thread as a looper, marking it as an 
    		application's main 
     *  looper. The main looper for your application is 
     		created by the Android environment,
     *  so you should never need to call this function yourself.
     * {@link #prepare()}
     */
 //由framework设置的UI程序的主消息循环，注意，这个主消息循环是不会主动退出的
//    
    public static final void prepareMainLooper() {
        prepare();
        setMainLooper(myLooper());
//判断主消息循环是否能退出....
//通过quit函数向looper发出退出申请
        if (Process.supportsProcesses()) {
            myLooper().mQueue.mQuitAllowed = false;
        }
    }

    private synchronized static void setMainLooper(Looper looper) {
        mMainLooper = looper;
    }
    
    /** Returns the application's main looper, 
    			which lives in the main thread of the application.
     */
    public synchronized static final Looper getMainLooper() {
        return mMainLooper;
    }

    /**
     *  Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
//消息循环，整个程序就在这里while了。
//这个是static函数喔！
    public static final void loop() {
        Looper me = myLooper();//从该线程中取出对应的looper对象
        MessageQueue queue = me.mQueue;//取消息队列对象..
        while (true) {
            Message msg = queue.next(); // might block取消息队列中的一个待处理消息..
            
            //是否需要退出？mRun是个volatile变量，跨线程同步的，应该是有地方设置它。
            //if (!me.mRun) {
            //    break;
            //}
            if (msg != null) {
                if (msg.target == null) {
                    // No target is a magic identifier for the quit message.
                    return;
                }
                if (me.mLogging!= null) me.mLogging.println(
                        ">>>>> Dispatching to " + msg.target + " "
                        + msg.callback + ": " + msg.what
                        );
                msg.target.dispatchMessage(msg);
                if (me.mLogging!= null) me.mLogging.println(
                        "<<<<< Finished to    " + msg.target + " "
                        + msg.callback);
                msg.recycle();
            }
        }
    }

    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
//返回和线程相关的looper
    public static final Looper myLooper() {
        return (Looper)sThreadLocal.get();
    }

    /**
     * Control logging of messages as they are processed by this Looper.  If
     * enabled, a log message will be written to <var>printer</var> 
     * at the beginning and ending of each message dispatch, identifying the
     * target Handler and message contents.
     * 
     * @param printer A Printer object that will receive log messages, or
     * null to disable message logging.
     */
//设置调试输出对象，looper循环的时候会打印相关信息，用来调试用最好了。
    public void setMessageLogging(Printer printer) {
        mLogging = printer;
    }
    
    /**
     * Return the {@link MessageQueue} object associated with the current
     * thread.  This must be called from a thread running a Looper, or a
     * NullPointerException will be thrown.
     */
    public static final MessageQueue myQueue() {
        return myLooper().mQueue;
    }
//创建一个新的looper对象，
//内部分配一个消息队列，设置mRun为true
    private Looper() {
        mQueue = new MessageQueue();
        mRun = true;
        mThread = Thread.currentThread();
    }

    public void quit() {
        Message msg = Message.obtain();
        // NOTE: By enqueueing directly into the message queue, the
        // message is left with a null target.  This is how we know it is
        // a quit message.
        mQueue.enqueueMessage(msg, 0);
    }

    /**
     * Return the Thread associated with this Looper.
     */
    public Thread getThread() {
        return mThread;
    }
    //后面就简单了，打印，异常定义等。
    public void dump(Printer pw, String prefix) {
        pw.println(prefix + this);
        pw.println(prefix + "mRun=" + mRun);
        pw.println(prefix + "mThread=" + mThread);
        pw.println(prefix + "mQueue=" + ((mQueue != null) ? 
        						mQueue : "(null"));
        if (mQueue != null) {
            synchronized (mQueue) {
                Message msg = mQueue.mMessages;
                int n = 0;
                while (msg != null) {
                    pw.println(prefix + "  Message " + n + ": " + msg);
                    n++;
                    msg = msg.next;
                }
                pw.println(prefix + "(Total messages: " + n + ")");
            }
        }
    }

    public String toString() {
        return "Looper{"
            + Integer.toHexString(System.identityHashCode(this))
            + "}";
    }

    static class HandlerException extends Exception {

        HandlerException(Message message, Throwable cause) {
            super(createMessage(cause), cause);
        }

        static String createMessage(Throwable cause) {
            String causeMsg = cause.getMessage();
            if (causeMsg == null) {
                causeMsg = cause.toString();
            }
            return causeMsg;
        }
    }
}
```

那怎么往这个消息队列中发送消息呢？？调用looper的static函数myQueue可以获得消息队列，这样你就可用自己往里边插入消息了。不过这种方法比较麻烦，这个时候handler类就发挥作用了。先来看看handler的代码，就明白了。

```
class Handler{
..........
//handler默认构造函数
public Handler() {
//这个if是干嘛用的暂时还不明白，涉及到java的深层次的内容了应该
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || 
            	klass.isMemberClass() || 
            		klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static 
                			or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
//获取本线程的looper对象
//如果本线程还没有设置looper，这回抛异常
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread 
                			that has not called Looper.prepare()");
        }
//无耻啊，直接把looper的queue和自己的queue搞成一个了
//这样的话，我通过handler的封装机制加消息的话，就相当于直接加到了looper的消息队列中去了
        mQueue = mLooper.mQueue;
        mCallback = null;
    }
//还有好几种构造函数，一个是带callback的，一个是带looper的
//由外部设置looper
    public Handler(Looper looper) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = null;
    }
// 带callback的，一个handler可以设置一个callback。如果有callback的话，
//凡是发到通过这个handler发送的消息，都有callback处理，相当于一个总的集中处理
//待会看dispatchMessage的时候再分析
public Handler(Looper looper, Callback callback) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
    }
//
//通过handler发送消息
//调用了内部的一个sendMessageDelayed
public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
//FT，又封装了一层，这回是调用sendMessageAtTime了
//因为延时时间是基于当前调用时间的，所以需要获得绝对时间传递给sendMessageAtTime
public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() 
        						+ delayMillis);
    }


public boolean sendMessageAtTime(Message msg, long uptimeMillis)
    {
        boolean sent = false;
        MessageQueue queue = mQueue;
        if (queue != null) {
//把消息的target设置为自己，然后加入到消息队列中
//对于队列这种数据结构来说，操作比较简单了
            msg.target = this;
            sent = queue.enqueueMessage(msg, uptimeMillis);
        }
        else {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
        }
        return sent;
    }
//还记得looper中的那个消息循环处理吗
//从消息队列中得到一个消息后，会调用它的target的dispatchMesage函数
//message的target已经设置为handler了，所以
//最后会转到handler的msg处理上来
//这里有个处理流程的问题
public void dispatchMessage(Message msg) {
//如果msg本身设置了callback，则直接交给这个callback处理了
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
//如果该handler的callback有的话，则交给这个callback处理了---相当于集中处理
          if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
           }
//否则交给派生处理,基类默认处理是什么都不干
            handleMessage(msg);
        }
    }
..........
}
```

 讲了这么多，该怎么创建和使用一个带消息循环的线程呢？
 
 ```
 //假设在onCreate中创建一个线程
//不花时间考虑代码的完整和严谨性了，以讲述原理为主。
....

... onCreate(...){

//难点是如何把android中的looper和java的thread弄到一起去。
//而且还要把随时取得这个looper用来创建handler
//最简单的办法就是从Thread派生一个
class ThreadWithMessageHandle extends Thread{
  //重载run函数
  Looper myLooper = null;
  run(){
  Looper.prepare();//将Looper设置到这个线程中
  myLooper = Looper.myLooper();
  Looper.loop();开启消息循环
}

 ThreadWithMessageHandle  threadWithMgs = new ThreadWithMessageHandle();
 threadWithMsg.start();
 Looper looper = threadWithMsg.myLooper;//
//这里有个问题.threadWithMgs中的myLooper可能此时为空
//需要同步处理一下
//或者像API文档中的那样，把handler定义到ThreadWithMessageHandle到去。
//外线程获得这个handler的时候仍然要注意同步的问题，因为handler的创建是在run中的
 Handler threadHandler = new Handler(looper);
 threadHandler.sendMessage(...)
}
}
...
```

好了，handler和looper的分析就都这了，其实原理挺简单的。

