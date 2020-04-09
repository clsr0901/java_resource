### 1. 创建线程

#### 继承Trhead类

```java
//通过继承Thread来创建线程
public class CreateThread extends Thread {
    //线程处理的具体业务
    @Override
    public void run() {
        System.out.println("create thread by 'extends Thread'");
        //this表示当前CreateThread对象
        System.out.println(this);
    }

    public static void main(String[] args) {
        //调用start方法来启动线程
        new CreateThread().start();
    }
}
```

#### 实现Runnable接口

```java
//通过实现Runnable接口来创建线程
public class CreateThreadByRunable implements Runnable {
	//线程处理的具体业务
    public void run() {
        System.out.println("create Thread 'by runable");
        //this表示当前CreateThreadByRunable对象
        System.out.println(this);
    }

    public static void main(String[] args) {
        //创建一个CreateThreadByRunable实例类
        Runnable runnable = new CreateThreadByRunable();
        //Thread可以接受一个Runnable对象
        //调用start方法来启动线程
        new Thread(runnable).start();
    }
}
```

实现Runnable可以解决继承Thread的缺点，Java中只能单继承，但可以多实现接口。啰嗦一句，多使用接口和组合的方式去替代继承，继承的侵入性太大。

#### 实现Callable接口

```java
//通过实现Callable接口来创建线程
public class CreateThreaByCallable implements Callable<String> {
    //线程处理的具体业务
    public String call() throws Exception {
        System.out.println("create thread 'by callable'");
        //this表示当前CreateThreaByCallable对象
        System.out.println(this);
        return "hello world";
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        //传入一个Callable对象来创建FutureTask对象
        FutureTask<String> stringFutureTask = new FutureTask<String>(new CreateThreaByCallable());
        //Thread可以接受一个FutureTask对象
        //调用start方法来启动线程
        new Thread(stringFutureTask).start();
        //阻塞并获取结果，可能抛出异常
        String s = stringFutureTask.get();
        System.out.println(s);
    }
}
```

如果需要返回值可以使用实现Callable接口来创建线程

### 线程的生命周期

![](https://user-gold-cdn.xitu.io/2020/3/8/170b9a0ef5d50d11?w=775&h=526&f=png&s=173157)

* New： 初始状态，一个Thread被创建，还没有调用start时的状态
* Runnable：运行状态，一个Thread被JVM执行时的状态，就绪状态也被笼统的叫做”运行“状态
* Blocked：表示线程阻塞于锁
* Waiting：等待状态，进入改状态需要其它线程做一些特殊动作（通知或中断）
* Timed Waiting：超时等待状态，该状态不同于Waiting，它是可以在指定的时间内自行返回
* Terminated：终止状态：表示当前线程已经执行完毕