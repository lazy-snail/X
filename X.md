---
title: X
categories: x
tags: [x]
---
[](https://it-blog-cn.com/blogs/interview/cache.html)

## SQL
### 分数分段统计
```sql
SELECT SUM(CASE WHEN score BETWEEN 90 AND 100 THEN 1 ELSE 0 END) AS A
      ,SUM(CASE WHEN score BETWEEN 80 AND 89 THEN 1 ELSE 0 END) AS B
      ,SUM(CASE WHEN score BETWEEN 70 AND 79 THEN 1 ELSE 0 END) AS C
      ,SUM(CASE WHEN score BETWEEN 60 AND 69 THEN 1 ELSE 0 END) AS D
      ,SUM(CASE WHEN score < 60 THEN 1 ELSE 0 END) AS E
FROM  student
```


## 正则
**邮箱** 只允许英文字母、数字、下划线、英文句号、以及中划线组成
```java
String pattern = "^[a-zA-Z0-9._-]+@[a-zA-Z0-9_-]+(\\.[a-zA-Z0-9_-]+)+$";
String content = "n-wei.shuai_07@gmail.com";
boolean isMatch = Pattern.matches(pattern, content);
```

## 单例
**双重校验**
```java
public class Singleton {
    // 修饰禁止重排序
    private volatile static Singleton instance;

    private Singleton() {
        //构造器必须私有  不然直接new就可以创建
    }

    public static Singleton getInstance() {
        //第一次判断，假设会有好多线程，如果doubleLock没有被实例化，那么就会到下一步获取锁,只有一个能获取到
        //如果已经实例化，那么直接返回，减少除了初始化时之外的所有锁获取等待过程
        if (instance == null) {
            //同步
            synchronized (Singleton.class) {
                //第二次判断是因为假设有两个线程A、B,两个同时通过了第一个if，然后A获取了锁，进入然后判断doubleLock是null，
                // 他就实例化了doubleLock，然后他出了锁，这时候线程B经过等待A释放的锁，B获取锁了，
                // 如果没有第二个判断，那么他还是会去new DoubleLock()，再创建一个实例，所以为了防止这种情况，需要第二次判断
                if (instance == null) {
                    //下面这句代码其实分为三步：
                    //1.开辟内存分配给这个对象
                    //2.初始化对象
                    //3.将内存地址赋给虚拟机栈内存中的doubleLock变量
                    //注意上面这三步，第2步和第3步的顺序是随机的，这是计算机指令重排序的问题
                    //假设有两个线程，其中一个线程执行下面这行代码，如果第三步先执行了，就会把没有初始化的内存赋值给doubleLock
                    //然后恰好这时候有另一个线程执行了第一个判断if(doubleLock == null)，然后就会发现doubleLock指向了一个内存地址
                    //这另一个线程就直接返回了这个没有初始化的内存，所以要防止第2步和第3步重排序
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**枚举**
```java
public enum Singleton {
    INSTANCE;

    public void doSomething() {
        // 实现具体逻辑
    }
}
```

**静态内部类**
```java
public class Singleton {
    // 静态内部类
    private static class SingletonHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    private Singleton() {}

    public static final Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

**饿汉模式**
```java
public final class Singleton {
    private static Singleton instance = new Singleton();// 自行创建实例
    private Singleton() {}
    public static Singleton getInstance() {// 通过该函数向整个系统提供实例
        return instance;
    }
}
```

**懒汉模式**
```java
public final class Singleton {
    private static Singleton instance = null;// 不实例化
    private Singleton() {}// 构造函数
    public static Singleton getInstance() {// 通过该函数向整个系统提供实例
        if (null == instance) {// 当 instance 为 null 时，则实例化对象，否则直接返回对象
            instance = new Singleton();// 实例化对象
        }
        return instance;// 返回已存在的对象
    }
}
```


## 多线程
### 两线程交替打印奇偶数
**wait/notify**
```java
public class WaitNotifyPrintOddEven {
    private static int count = 0;
    private static final Object lock = new Object();
    
    public static void main(String[] args) {
        new Thread(new TurningRunner(),"偶数").start();
        new Thread(new TurningRunner(),"奇数").start();
    }
    //拿到锁，就打印,一旦打印完唤醒其他线程就休眠
    static class TurningRunner implements Runnable {
        @Override
        public void run() {
            while (count <= 100) {
                synchronized (lock) {
                    System.out.println(Thread.currentThread().getName() + ":" + count++);
                    lock.notify();
                    if (count <= 100) {
                        try {
                            //如果任务没结束，唤醒其他线程，自己休眠
                            lock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }
    }
}
```

### N线程顺序打印1-100
```java
public class MultiThread {
    private static final int N = 10; // 线程数
    private static final int M = 100; // 打印最大值

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < N; i++) {
            MyThread myThread = new MyThread(i);
            myThread.join();
            myThread.start();
        }
    }

    static class MyThread extends Thread {
        private static final AtomicInteger NUM = new AtomicInteger(0);
        private static volatile int now; // 现在该哪个线程打印了，递增
        /**
         * 当前线程的id，不变
         */
        private final int id;

        public MyThread(int id) {
            this.id = id;
            this.setName("线程" + id);
        }

        @Override
        public void run() {
            while (NUM.get() < M) {
                if (id == now) {
                    System.out.println(Thread.currentThread().getName() + ": " + NUM.getAndIncrement());
                    now = (now + 1) % N;
                }
            }
        }
    }
}
```

### 三线程轮流打印1-100
**利用synchronized实现（不能实现精准唤醒）**
```java
public class Test {
    int num = 0;
    Object o = new Object();

    private void print(int targetNum) throws InterruptedException {
        while (true) {
            synchronized (o) {
                while (num % 3 != targetNum) {
                    o.wait();
                }
                if (num >= 100) {
                    break;
                }
                num++;
                System.out.println(Thread.currentThread().getName() + ": " + num);
                o.notifyAll();

            }
        }
    }

    public static void main(String[] args) {
        Test main = new Test();
        new Thread(() -> {
            try {
                main.print(0);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "A").start();

        new Thread(() -> {
            try {
                main.print(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "B").start();

        new Thread(() -> {
            try {
                main.print(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "C").start();
    }
}
```

**利用ReentrantLock的Condition实现精准唤醒**
```java
public class Test {
    int num;
    static ReentrantLock lock = new ReentrantLock();
    static Condition c1 = lock.newCondition();
    static Condition c2 = lock.newCondition();
    static Condition c3 = lock.newCondition();

    private void printABC(int targetNum, Condition currentThread, Condition nextThread) {
        while (true) {
            lock.lock();
            try {
                while (num % 3 != targetNum) {
                    currentThread.await();
                }
                if (num >= 100) {
                    break;
                }
                num++;
                System.out.println(Thread.currentThread().getName() + ": " + num);
                nextThread.signal();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }

        }

    }

    public static void main(String[] args) {
        Test test = new Test();
        new Thread(() -> {
            test.printABC(0, c1, c2);
        }, "A").start();

        new Thread(() -> {
            test.printABC(1, c2, c3);
        }, "B").start();

        new Thread(() -> {
            test.printABC(2, c3, c1);
        }, "C").start();
    }
}
```

### 三线程轮流打印ABC
**fork join**
```java
public class Test {
    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 10; i++) {
            Thread t1 = new Thread(new printABC(null), "A");
            Thread t2 = new Thread(new printABC(t1), "B");
            Thread t3 = new Thread(new printABC(t2), "C");
            t1.start();
            t2.start();
            t3.start();
            Thread.sleep(10); //这里是要保证只有t1、t2、t3为一组，进行执行才能保证t1->t2->t3的执行顺序。
        }
    }

    static class printABC implements Runnable {
        private Thread beforeThread;

        public printABC(Thread beforeThread) {
            this.beforeThread = beforeThread;
        }

        @Override
        public void run() {
            if (beforeThread != null) {
                try {
                    beforeThread.join();
                    System.out.print(Thread.currentThread().getName());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            } else {
                System.out.print(Thread.currentThread().getName());
            }
        }
    }
}
```

### N个生产者M个消费者问题
```java
public class Main {
    public static void main(String[] args) {
        Resource resource = new Resource(10);
        new Producer(3, resource).start();
        new Producer(3, resource).start();
        new Producer(3, resource).start();

        new Consumer(2, resource).start();
        new Consumer(2, resource).start();
    }

    // 共享资源
    public static class Resource {
        LinkedList<String> commonList;
        int capacity = 0;

        public Resource(int capacity) {
            this.capacity = capacity;
            this.commonList = new LinkedList<>();
        }

        public void addLast(String item) {
            commonList.addLast(item);
        }

        public void removeFirst() {
            commonList.removeFirst();
        }

        public int size() {
            return commonList.size();
        }

        public boolean isFull() {
            return size() == this.capacity;
        }
    }

    // 生产者
    public static class Producer extends Thread {
        private int duration;
        private Resource resource;

        public Producer(int duration, Resource resource) {
            this.duration = duration;
            this.resource = resource;
        }

        @Override
        public void run() {
            while (true) {
                try {
                    synchronized (resource) {
                        if (!resource.isFull()) {
                            resource.addLast("P_" + Thread.currentThread().getName());
                            // notify/notifyAll() 方法，唤醒一个或多个正处于等待状态的线程
                            resource.notifyAll();
                        } else {
                            System.out.println("resource is full, waiting...");
                            // wait()使当前线程阻塞，前提是必须先获得锁配合synchronized 关键字使用
                            resource.wait();
                        }
                    }
                    Thread.sleep(1000 * duration);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    // 消费者
    public static class Consumer extends Thread {
        private int duration;
        private Resource resource;

        public Consumer(int duration, Resource resource) {
            this.duration = duration;
            this.resource = resource;
        }

        @Override
        public void run() {
            while (true) {
                try {
                    synchronized (resource) {
                        // 共享区有数据才能取数据
                        if (resource.size() > 0) {
                            System.out.println("Consumed_resource.size() = " + resource.size());
                            resource.removeFirst();
                            // 消费数据后唤醒生产者
                            resource.notifyAll();
                        } else {
                            System.out.println("resource is empty, waiting...");
                            // wait()使当前线程阻塞，前提是 必须先获得锁，配合synchronized 关键字使用
                            resource.wait();
                        }
                    }
                    Thread.sleep(1000 * duration);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

## 设计模式
https://refactoringguru.cn/design-patterns/proxy
