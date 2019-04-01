AsyncTask虽说是轻量级的但用起来也不是很方便，而且还有很多弊端
1、每次使用时，都需要创建一个子类
AsyncTask是一个抽象类，无法通过new关键字直接创建一个实例对象，所以我们要想使用它，就必须创建一个子类去继承它，并指明它的三个泛型<Params,Progress,Result>。然后实例化子类对象调用execute()方法。
示例如下
```java
class MyAsycTask extends AsyncTask<String, Integer, Boolean> {
        @Override
        protected void onPreExecute() {
            progressDialog.show();//显示进度框
        }

        @Override
        protected Boolean doInBackground(String... integers) {
            publishProgress(currentProgress()); //向UI线程发送当前进度
            return doDownload();    //向ui线程反馈下载成功与否
        }

        @Override
        protected void onPostExecute(Boolean result) {
            progressDialog.dismiss(); //隐藏进度框
            //TODO:根据result判断下载成功还是失败
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
            progressDialog.setMessage("当前下载进度：" + values[0] + "%");//在UI线程中展示当前进度
        }

    }
```
然后再在需要使用此异步任务的地方进行开启
```java
public void startTask() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {       //根据不同的api采用并行模式进行开启
            new MyAsycTask().executeOnExecutor(THREAD_POOL_EXECUTOR,"参数");
        } else {
            new MyAsycTask().execute("参数");
        }
    }
```
2、生命周期不受Activity的约束
关于AsyncTask的使用有一个很容易被人忽略的地方，那就是在Activity中开启的AsyncTask并不会随着Activity的销毁而销毁。即使Activity已经销毁甚至所属的应用进程已经销毁，AsyncTask还是会一直执行doInBackground(Params... params)方法，直到方法执行完成，然后根据cancel(boolean mayInterruptIfRunning)是否调用，回调onCancelled(Result result)或onPostExecute(Result result)方法。
3、cancel(boolean mayInterruptIfRunning)方法调用不当会失效。
如果的doInBackground()中有不可打断的方法则cancel方法可能会失效，比如BitmapFactory.decodeStream()， IO等操作，也就是即使调用了cancel方法，onPostExecute(Result result)方法仍然有可能被回调。
4、容易导致内存泄漏，Activity不被回收
使用AsyncTask时，经常出现的情况是，在Activity中使用非静态匿名内部AsyncTask类，由于Java内部类的特点，AsyncTask内部类会持有外部类Activity的隐式引用。由于AsyncTask的生命周期可能比Activity的长，当Activity进行销毁AsyncTask还在执行时，由于AsyncTask持有Activity的引用，导致Activity对象无法回收，进而产生内存泄露。
5、Activity被回收，操作UI时出现空指针异常
如果在Activity中使用静态AsynTask子类，依然无法高枕无忧，虽然静态AsyncTask子类中不会持有Activity的隐式引用。但是AsyncTask的生命周期依然可能比Activity的长，当Activity进行销毁，AsyncTask执行回调onPostExecute(Result result)方法时，如果在onPostExecute(Result result)中进行了UI操作，UI对象会随着Activity的回收而被回收，导致UI操作报空指针异常。
6、onPostExecute()回调UI线程不起作用
如果在开启AsyncTask后，Activity重新创建重新创建了例如屏幕旋转等，还在运行的AsyncTask会持有一个Activity的非法引用即之前的Activity实例，会导致AsyncTask执行后的数据丢失而且之前的Activity对象不被回收。
7、多个异步任务同时开启时，存在串行并行问题。
当多个异步任务调用方法execute()同时开启时，api的不同会导致串行或并行的执行方式的不同。
在1.6(Donut)之前，多个异步任务串行执行
从1.6到2.3(Gingerbread)，多个异步任务并行执行
3.0（Honeycomb）到现在，如果调用方法execute()开启是串行，调用executeOnExecutor(Executor)是并行。
AnsycTask虽然是个轻量级的异步操作类，方便了子线程与UI线程的数据传递，但是使用不当又会产生很多问题。所以有必要对其进行进一步的封装，以使它在方便我们调用的同时还能消除弊端。


