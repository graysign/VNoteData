单例模式确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个实例单例模式。单例模式只应在有真正的“单一实例”的需求时才可使用。
```java
class Singleton {
    private static Singleton sing;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (sing == null) {
            sing = new Singleton();
        }

        return sing;
    }
}


public class Test {
    public static void main(String[] args) {
        Singleton sing = Singleton.getInstance();
        Singleton sing2 = Singleton.getInstance();
        System.out.println(sing);
        System.out.println(sing2);
    }
}

```

不过这样的单例模式似乎有些问题，在多线程并发的情况下，if(sing=null)的判断上还是有可能出现两个单例的情况，所以为避免万一，最好还是改成：  

```java
class Singleton {
    private static Singleton sing = new Singleton();

    private Singleton() {
    }

    public static Singleton getInstance() {
        return sing;
    }
}

```
还有更牛X的方法进行单例~
使用枚举：

```java
    enum Singleton {sing;

    //方法
    public void test() {
        System.out.println("singleton");
    }
}

```

因为枚举默认每个都是单例
以为结束了吗？不~使用静态内部类：
```java
class Singleton {
    private Singleton() {
    }

    public static Singleton getInstance() {
        return SingletonInner.sing;
    }

    private static class SingletonInner {
        private static Singleton sing = new Singleton();
    }
}

```
这样用的好处在于可以在类在装载的时候并没有创建这个单例，而在真正使用的时候才去调用内部类去创建这个单例