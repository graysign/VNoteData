代理模式是常用的java设计模式，他的特征是代理类与委托类有同样的接口，代理类主要负责为委托类预处理消息、过滤消息、把消息转发给委托类，以及事后处理消息等。代理类与委托类之间通常会存在关联关系，一个代理类的对象与一个委托类的对象关联，代理类的对象本身并不真正实现服务，而是通过调用委托类的对象的相关方法，来提供特定的服务。 按照代理的创建时期，代理类可以分为两种。   

* 静态代理：由程序员创建或特定工具自动生成源代码，再对其编译。在程序运行前，代理类的.class文件就已经存在了。 
* 动态代理：在程序运行时，运用反射机制动态创建而成。   
###### 静态代理

1. 
```java
package net.battier.dao;  
  
/** 
 * 定义一个账户接口 
 *  
 * @author Administrator 
 *  
 */  
public interface Count {  
    // 查看账户方法  
    public void queryCount();  
  
    // 修改账户方法  
    public void updateCount();  
  
}  
```
2. 
```java
 package net.battier.dao.impl;  
  
import net.battier.dao.Count;  
  
/** 
 * 委托类(包含业务逻辑) 
 *  
 * @author Administrator 
 *  
 */  
public class CountImpl implements Count {  
  
    @Override  
    public void queryCount() {  
        System.out.println("查看账户方法...");  
  
    }  
  
    @Override  
    public void updateCount() {  
        System.out.println("修改账户方法...");  
  
    }  
  
}  
  
、CountProxy.java  
package net.battier.dao.impl;  
  
import net.battier.dao.Count;  
  
/** 
 * 这是一个代理类（增强CountImpl实现类） 
 *  
 * @author Administrator 
 *  
 */  
public class CountProxy implements Count {  
    private CountImpl countImpl;  
  
    /** 
     * 覆盖默认构造器 
     *  
     * @param countImpl 
     */  
    public CountProxy(CountImpl countImpl) {  
        this.countImpl = countImpl;  
    }  
  
    @Override  
    public void queryCount() {  
        System.out.println("事务处理之前");  
        // 调用委托类的方法;  
        countImpl.queryCount();  
        System.out.println("事务处理之后");  
    }  
  
    @Override  
    public void updateCount() {  
        System.out.println("事务处理之前");  
        // 调用委托类的方法;  
        countImpl.updateCount();  
        System.out.println("事务处理之后");  
  
    }  
  
}  
```
3. 
```java
package net.battier.test;  
  
import net.battier.dao.impl.CountImpl;  
import net.battier.dao.impl.CountProxy;  
  
/** 
 *测试Count类 
 *  
 * @author Administrator 
 *  
 */  
public class TestCount {  
    public static void main(String[] args) {  
        CountImpl countImpl = new CountImpl();  
        CountProxy countProxy = new CountProxy(countImpl);  
        countProxy.updateCount();  
        countProxy.queryCount();  
  
    }  
}  
```
观察代码可以发现每一个代理类只能为一个接口服务，这样一来程序开发中必然会产生过多的代理，而且，所有的代理操作除了调用的方法不一样之外，其他的操作都一样，则此时肯定是重复代码。解决这一问题最好的做法是可以通过一个代理类完成全部的代理功能，那么此时就必须使用动态代理完成。   

###### 动态代理：   
JDK动态代理中包含一个类和一个接口：     
InvocationHandler接口：   
```java
public interface InvocationHandler { 
public Object invoke(Object proxy,Method method,Object[] args) throws Throwable; 
} 
//参数说明： 
//Object proxy：指被代理的对象。 
//Method method：要调用的方法 
//Object[] args：方法调用时所需要的参数 
//可以将InvocationHandler接口的子类想象成一个代理的最终操作类，替换掉ProxySubject。 
```
Proxy类：   
Proxy类是专门完成代理的操作类，可以通过此类为一个或多个接口动态地生成实现类，此类提供了如下的操作方法：   
```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, 
InvocationHandler h) throws IllegalArgumentException 
//参数说明： 
//ClassLoader loader：类加载器 
//Class<?>[] interfaces：得到全部的接口 
//InvocationHandler h：得到InvocationHandler接口的子类实例 
```

Ps:类加载器   
在Proxy类中的newProxyInstance（）方法中需要一个ClassLoader类的实例，ClassLoader实际上对应的是类加载器，在Java中主要有一下三种类加载器;   
1. Booststrap ClassLoader：此加载器采用C++编写，一般开发中是看不到的；
2. Extendsion ClassLoader：用来进行扩展类的加载，一般对应的是jre\lib\ext目录中的类;
3. AppClassLoader：(默认)加载classpath指定的类，是最常使用的是一种加载器。  

与静态代理类对照的是动态代理类，动态代理类的字节码在程序运行时由Java反射机制动态生成，无需程序员手工编写它的源代码。动态代理类不仅简化了编程工作，而且提高了软件系统的可扩展性，因为Java 反射机制可以生成任意类型的动态代理类。java.lang.reflect 包中的Proxy类和InvocationHandler 接口提供了生成动态代理类的能力。   
动态代理示例:   
1.  
```java
package net.battier.dao;  
  
public interface BookFacade {  
    public void addBook();  
}  
```
2.  
```java
package net.battier.dao.impl;  
  
import net.battier.dao.BookFacade;  
  
public class BookFacadeImpl implements BookFacade {  
  
    @Override  
    public void addBook() {  
        System.out.println("增加图书方法。。。");  
    }  
  
}  
  
、BookFacadeProxy.java  
  
package net.battier.proxy;  
  
import java.lang.reflect.InvocationHandler;  
import java.lang.reflect.Method;  
import java.lang.reflect.Proxy;  
  
/** 
 * JDK动态代理代理类 
 *  
 * @author student 
 *  
 */  
public class BookFacadeProxy implements InvocationHandler {  
    private Object target;  
    /** 
     * 绑定委托对象并返回一个代理类 
     * @param target 
     * @return 
     */  
    public Object bind(Object target) {  
        this.target = target;  
        //取得代理对象  
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),  
                target.getClass().getInterfaces(), this);   //要绑定接口(这是一个缺陷，cglib弥补了这一缺陷)  
    }  
  
    @Override  
    /** 
     * 调用方法 
     */  
    public Object invoke(Object proxy, Method method, Object[] args)  
            throws Throwable {  
        Object result=null;  
        System.out.println("事物开始");  
        //执行方法  
        result=method.invoke(target, args);  
        System.out.println("事物结束");  
        return result;  
    }  
  
}  
```
3.   
```java
package net.battier.test;  
  
import net.battier.dao.BookFacade;  
import net.battier.dao.impl.BookFacadeImpl;  
import net.battier.proxy.BookFacadeProxy;  
  
public class TestProxy {  
  
    public static void main(String[] args) {  
        BookFacadeProxy proxy = new BookFacadeProxy();  
        BookFacade bookProxy = (BookFacade) proxy.bind(new BookFacadeImpl());  
        bookProxy.addBook();  
    }  
  
}  
```
但是，JDK的动态代理依靠接口实现，如果有些类并没有实现接口，则不能使用JDK代理，这就要使用cglib动态代理了。  

###### Cglib动态代理  
JDK的动态代理机制只能代理实现了接口的类，而不能实现接口的类就不能实现JDK的动态代理，cglib是针对类来实现代理的，他的原理是对指定的目标类生成一个子类，并覆盖其中方法实现增强，但因为采用的是继承，所以不能对final修饰的类进行代理。  
示例   
1.   
```java
package net.battier.dao;  
  
public interface BookFacade {  
    public void addBook();  
}  
```
2.  
```java
package net.battier.dao.impl;  
  
/** 
 * 这个是没有实现接口的实现类 
 *  
 * @author student 
 *  
 */  
public class BookFacadeImpl1 {  
    public void addBook() {  
        System.out.println("增加图书的普通方法...");  
    }  
}  
```
3.  
```java
package net.battier.proxy;  
  
import java.lang.reflect.Method;  
  
import net.sf.cglib.proxy.Enhancer;  
import net.sf.cglib.proxy.MethodInterceptor;  
import net.sf.cglib.proxy.MethodProxy;  
  
/** 
 * 使用cglib动态代理 
 *  
 * @author student 
 *  
 */  
public class BookFacadeCglib implements MethodInterceptor {  
    private Object target;  
  
    /** 
     * 创建代理对象 
     *  
     * @param target 
     * @return 
     */  
    public Object getInstance(Object target) {  
        this.target = target;  
        Enhancer enhancer = new Enhancer();  
        enhancer.setSuperclass(this.target.getClass());  
        // 回调方法  
        enhancer.setCallback(this);  
        // 创建代理对象  
        return enhancer.create();  
    }  
  
    @Override  
    // 回调方法  
    public Object intercept(Object obj, Method method, Object[] args,  
            MethodProxy proxy) throws Throwable {  
        System.out.println("事物开始");  
        proxy.invokeSuper(obj, args);  
        System.out.println("事物结束");  
        return null;  
  
  
    }  
  
}  
```
4.   
```java
package net.battier.test;  
  
import net.battier.dao.impl.BookFacadeImpl1;  
import net.battier.proxy.BookFacadeCglib;  
  
public class TestCglib {  
      
    public static void main(String[] args) {  
        BookFacadeCglib cglib=new BookFacadeCglib();  
        BookFacadeImpl1 bookCglib=(BookFacadeImpl1)cglib.getInstance(new BookFacadeImpl1());  
        bookCglib.addBook();  
    }  
}  
```