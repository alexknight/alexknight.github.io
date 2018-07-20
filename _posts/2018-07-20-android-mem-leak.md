---
title: 记一次Android内存泄漏的问题分析
category: Android
---

最近看了下`MonitoSDK`的代码，试用了下里面的`Sample App`，然后在使用”测试数据库”时，发现存在部分内存泄漏的情况，`leakcanary`直接弹出提示，可以看到`MonitorDBActivity`的`instance`导致泄漏。

## 一.`Leak Canary`报错效果图
![mem-leak](/img/postimg/leak1.png)


## 二.过程分析

~~~java
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_db);

        ButterKnife.bind(this);
    }

    @OnClick(R.id.inputData)
    public void inputData() {

        EditText input = (EditText) findViewById(R.id.inputTestData);
        int num = Integer.valueOf(input.getEditableText().toString());
        ErrorReporter reporter = ACRA.getErrorReporter();
        CrashReportDataFactory factory = null;
        try {
            factory = Reflect.on(reporter).get("crashReportDataFactory");
        } catch (ReflectException e) {
            e.printStackTrace();
        }
        for (int i = 0; i < num; i++) {
            final ReportBuilder builder = new ReportBuilder();
            builder.exception(new Exception(i + ""));
            final CrashReportData crashReportData = factory.createCrashData(builder);
            MLog log = null;
            try {
                log = new MLog(crashReportData.toJSON().toString());
            } catch (JSONReportBuilder.JSONReportException e) {
                e.printStackTrace();
            }
            MLogStoreMgr.getInstance(this).add(log);
        }
    }
~~~

`log`相关操作只有`MLogStoreMgr.getInstance(this).add(log)`;这一行代码，继续`debug`，可以看到最后底层执行的是

~~~java
if(!UploadTask.isRunning()) {
            TaskExecutor.getInstance().postDelayed(1, new UploadTask() {
                public void onUploadExcuted() {
                    if(UploadEngine.this.bRunning) {
                        UploadEngine.this.calNextInterval();
                        if(TaskExecutor.getInstance().hasCallbacks(1)) {
                            TaskExecutor.getInstance().removeCallbacks(1);
                        }

                        if(!UploadTask.isRunning()) {
                            TaskExecutor.getInstance().postDelayed(1, this, UploadEngine.this.mPeriod);
                        }
                    }

                }

                public void deleteError() {
                }
            }, this.mPeriod);
        }
~~~

没错，就是这个`postDelayed`导致的。因为这个方法，导致主线程`MainActivity`的引用无法被释放，直到消息被`looper`处理掉。要解决这种问题，思路就是避免使用非静态内部类，继承`Handler`时，要么是放在单独的类文件中，要么就是使用静态内部类。因为静态的内部类不会持有外部类的引用，所以不会导致外部类实例的内存泄露。当你需要在静态内部类中调用外部的`Activity`时，我们可以使用弱引用来处理。

假如把耗时操作直接放在子线程中用`Handler`处理呢？这样也是不行的，如果在Handler中设置了延时操作，则调用线程也会堵塞。每个`Handler`对象都会绑定一个`Looper`对象，每个`Looper`对象对应一个消息队列（`MessageQueue`）。如果在创建`Handler`时不指定与其绑定的`Looper`对象，系统默认会将当前线程的`Looper`绑定到该`Handler`上。
在主线程中，可以直接使用`new Handler()`创建`Handler`对象，其将自动与主线程的`Looper`对象绑定；在非主线程中直接这样创建`Handler`则会报错，因为`Android`系统默认情况下非主线程中没有开启`Looper`，而`Handler`对象必须绑定`Looper`对象。
所以修改后的代码如下：

~~~java
    @OnClick(R.id.inputData)
    public void inputData() {
        ...
        for (int i = 0; i < num; i++) {
            ...
            logs.add(log);
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Looper.prepare();
                    new Handler(Looper.getMainLooper()).post(new Runnable() {
                        @Override
                        public void run() {
                            for (MLog log: logs){
                                MLogStoreMgr.getInstance(MonitorDBActivity.this).add(log);
                            }
                            Toast.makeText(MonitorDBActivity.this,"写入数据",Toast.LENGTH_SHORT).show();
                        }
                    });
                    Looper.loop();
                }
            }).start();
        }
        Toast.makeText(this,"开始发送消息",Toast.LENGTH_SHORT).show();
~~~

后续：这种问题算是出现的比较多的问题，一定要注意两点：
- 尽量不在主线程里做耗时操作；
- 不用忽视`Android Lint`的提示，`Handler`的内存泄漏用静态代码检查是可以查出来的。