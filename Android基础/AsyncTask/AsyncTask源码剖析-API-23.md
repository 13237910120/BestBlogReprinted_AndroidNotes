# AsyncTask源码剖析(API 23)

来源:[Spark Yuan](http://sparkyuan.me/2016/03/23/AsyncTask%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90(API%2023))

Android的UI是线程不安全的，想在子线程中更新UI就必须使用Android的异步操作机制，直接在主线程中更新UI会导致程序崩溃。

Android的异步操作主要有两种，AsyncTask和Handler。AsyncTask是一个轻量的异步类，简单、可控。本文主要结合API 23的源码讲解一下AsyncTask到底是什么。

## 基本用法
声明：Android不同API版本中同一个类的实现方法可能会有不同，本文是基于最新的API 23的源码进行讲解的。

```
public abstract class AsyncTask<Params, Progress, Result>
```

* Params：执行时传入的参数
* Progress：后台任务的执行进度
* Result：返回值

AsyncTask是个抽象类，所以需要自己定义一个类继承他，比如

```
class MyAsyncTask extends AsyncTask<Void, Integer, Boolean>
```


### AsyncTask的执行过程

* execute(Params… params)，执行异步任务。
* onPreExecute()，在execute(Params… params)被调用后执行，界面上的初始化操作，比如显示一个进度条对话框等。
* doInBackground(Params… params)，在onPreExecute()完成后执行，用于执行较为费时的操作，如果AsyncTask的第三个泛型参数指定的是Void，就可以不返回任务执行结果。在执行过程中可以调用publishProgress(Progress… values)来更新进度信息。注意，此方法中不可以进行UI操作。
* onProgressUpdate(Progress… values)，调用publishProgress(Progress… values)时，此方法被执行，将进度信息更新到UI组件上。
* onPostExecute(Result result)，当后台操作结束时，此方法将会被调用，计算结果将做为参数传递到此方法中，可以利用返回的数据来进行一些UI操作，比如说提醒任务执行的结果，以及关闭掉进度条对话框等。

## 示例

新建一个Activity，一个Button和一个ProgressBar，点击Button启动一个AsyncTask并实时更新ProgressBar的状态。

### MyAsyncTask

```
class MyAsyncTask extends AsyncTask<Void, Integer, Boolean> {

        @Override
        protected void onPreExecute() {
            super.onPreExecute();
        }

        @Override
        protected void onPostExecute(Boolean aBoolean) {
            super.onPostExecute(aBoolean);
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
            super.onProgressUpdate(values);
            progressBar.setProgress(values[0]);
        }

        @Override
        protected Boolean doInBackground(Void... params) {
            for (int i = 0; i < 100; i++) {
                //调用publishProgress,触发onProgressUpdate方法
                publishProgress(i);
                try {
                    Thread.sleep(300);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            return true;
        }

        @Override
        protected void onCancelled() {
            super.onCancelled();
            progressBar.setProgress(0);
        }
    }
```

### Button的Click方法

```
startAsyncBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                myAsyncTask = new MyAsyncTask();
                myAsyncTask.execute();
            }
        });
```

## 源码剖析

通过上面的例子可以发现，AsyncTask使用起来很简单，很方便的就可以在主线程中新建一个子线程进行UI的更新等操作。但是他的实现并不像使用起来那么简单，下面就是对AsyncTask的源码进行剖析。

### AsyncTask的构造函数

```
public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```

在构造函数中只做了两件事，初始化mWorker和mFuture两个变量。mWorker是一个Callable对象，mFutre是一个FutureTask对象。execute()时会用到。

### execute(Params… params)

```
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
       return executeOnExecutor(sDefaultExecutor, params);
   }
```


只有一行代码，调用了executeOnExecutor方法，sDefaultExecutor实际上是一个串行的线程池，一个进程中所有的AsyncTask全部在这个串行的线程池中排队执行。executeOnExecutor源码如下。

```
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;
        onPreExecute();
        mWorker.mParams = params;
        exec.execute(mFuture);
        return this;
    }
```

可以看到在这个方法里调用了onPreExecute()，接下来执行exec.execute(mFuture)下面分析一一下线程池的执行过程。

```
private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```

系统先把AsyncTask的Params参数封装为FutureTask对象，FutureTask是一个并发类，这里它相当于Runnable；接着将FutureTask交给SerialExecutor的execute方法，它先把FutureTask插入到任务队列tasks中，如果这个时候没有正在活动的AsyncTask任务，那么就会执行下一个AsyncTask任务，同时当一个AsyncTask任务执行完毕之后，AsyncTask会继续执行其他任务直到所有任务都被执行为止。从这里就可以看出，默认情况下，AsyncTask是串行执行的

看一下AsyncTask的构造函数，mFuture构造时是把mWork作为参数传进去的，mFuture的run方法会调用mWork的call()方法，因此call()最终会在线程池中执行。call()中调用了doInBackground()并把返回结果给了postResult。

```
private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```

可以看到在postResult中通过getHandler()获得一个Handler。

```
private static Handler getHandler() {
      synchronized (AsyncTask.class) {
          if (sHandler == null) {
              sHandler = new InternalHandler();
          }
          return sHandler;
      }
  }
```

查看getHandler源码，可以发现getHandler返回的是一个InternalHandler，再来看看InternalHandler的源码。

```
private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```

看到这里已经豁然开朗了，InternalHandler是一个继承Handler的类，在他的handleMessage()方法中对msg进行了判断，如果是MESSAGE_POST_RESULT就执行finish()，如果是MESSAGE_POST_PROGRESS，就执行onProgressUpdate()。
MESSAGE_POST_PROGRESS消息是在publishProgress里发出的，详情见源码。

```

protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```

### finish()

```
private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```

如果当前任务被取消，就调用onCancelled()方法，如果没有调用onPostExecute()。

## 注意事项

* AsyncTask的类必须在主线程中加载，这个过程在Android 4.1及以上版本中已经被系统自动完成。
* AsyncTask对象必须在主线程中创建，execute方法必须在UI线程中调用。
* 一个AsyncTask对象只能执行一次，即只能调用一次execute方法，否则会报运行时异常。

## 各个版本的区别

在Android 1.6之前，AsyncTask是串行执行任务的，Android 1.6的时候AsyncTask开始采用线程池并行处理任务，但是从Android 3.0开始，为了避免AsyncTask带来的并发错误，AsyncTask又采用一个线程来串行执行任务。尽管如此，在Android 3.0以及后续版本中，我们可以使用AsyncTask的executeOnExecutor方法来并行执行任务。但是这个方法是Android 3.0新添加的方法，并不能在低版本上使用。

## 总结

整个AsyncTask的源码已经剖析完了，在分析完真个源码后可以发现，AsyncTask并没有什么神秘的，他的本质就是Handler。

我们现在已经知道了AsyncTask如何使用，各个方法会在什么时候调用，有什么作用，相互之间有什么联系，相信大家以后在遇到AsyncTask的任何问题都不会再害怕了，因为AsyncTask的整个源码都翻了个底朝天，还有什么好怕的呢。