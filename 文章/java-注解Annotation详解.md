注解相当于一种标记，在程序中加了注解就等于为程序打上了某种标记，没加，则等于没有某种标记，`以后，javac编译器，开发工具和其他程序可以用反射来了解你的类及各种元素上有无何种标记，看你有什么标记，就去干相应的事`。标记可以加在包，类，字段，方法，方法的参数以及局部变量上  
自定义注解及其应用  

* 定义一个最简单的注解  
```java
public @interface MyAnnotation {
    //......
}
```
* 把注解加在某个类上：  
```java
@MyAnnotation
public class AnnotationTest{
    //......
}
```
以下为模拟案例, 自定义注解@MyAnnotation  

```java
package com.ljq.test;

 import java.lang.annotation.ElementType;
 import java.lang.annotation.Retention;
 import java.lang.annotation.RetentionPolicy;
 import java.lang.annotation.Target;

 /**
 * 定义一个注解
 * 
 * 
 * @author jiqinlin
 *
 */
 //Java中提供了四种元注解，专门负责注解其他的注解，分别如下

 //@Retention元注解，表示需要在什么级别保存该注释信息（生命周期）。可选的RetentionPoicy参数包括：
 //RetentionPolicy.SOURCE: 停留在java源文件，编译器被丢掉
 //RetentionPolicy.CLASS：停留在class文件中，但会被VM丢弃（默认）
 //RetentionPolicy.RUNTIME：内存中的字节码，VM将在运行时也保留注解，因此可以通过反射机制读取注解的信息

 //@Target元注解，默认值为任何元素，表示该注解用于什么地方。可用的ElementType参数包括
 //ElementType.CONSTRUCTOR: 构造器声明
 //ElementType.FIELD: 成员变量、对象、属性（包括enum实例）
 //ElementType.LOCAL_VARIABLE: 局部变量声明
 //ElementType.METHOD: 方法声明
 //ElementType.PACKAGE: 包声明
 //ElementType.PARAMETER: 参数声明
 //ElementType.TYPE: 类、接口（包括注解类型)或enum声明

 //@Documented将注解包含在JavaDoc中

 //@Inheried允许子类继承父类中的注解

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
 public @interface MyAnnotation {
    //为注解添加属性
     String color();
    String value() default "我是林计钦"; //为属性提供默认值
     int[] array() default {1, 2, 3}; 
    Gender gender() default Gender.MAN; //添加一个枚举
     MetaAnnotation metaAnnotation() default @MetaAnnotation(birthday="我的出身日期为1988-2-18");
    //添加枚举属性
 
}
```
注解测试类AnnotationTest  
```java
package com.ljq.test;

 /**
 * 注解测试类
 * 
 * 
 * @author jiqinlin
 *
 */
 //调用注解并赋值
 @MyAnnotation(metaAnnotation=@MetaAnnotation(birthday = "我的出身日期为1988-2-18"),color="red", array={23, 26})
 public class AnnotationTest {

    public static void main(String[] args) {
        //检查类AnnotationTest是否含有@MyAnnotation注解
         if(AnnotationTest.class.isAnnotationPresent(MyAnnotation.class)){
            //若存在就获取注解
             MyAnnotation annotation=(MyAnnotation)AnnotationTest.class.getAnnotation(MyAnnotation.class);
            System.out.println(annotation);
            //获取注解属性
             System.out.println(annotation.color()); 
            System.out.println(annotation.value());
            //数组
             int[] arrs=annotation.array();
            for(int arr:arrs){
                System.out.println(arr);
            }
            //枚举
             Gender gender=annotation.gender();
            System.out.println("性别为："+gender);
            //获取注解属性
             MetaAnnotation meta=annotation.metaAnnotation();
            System.out.println(meta.birthday());
        }
    }
}
```

枚举类Gender，模拟注解中添加枚举属性  
```java
package com.ljq.test;
 /**
 * 枚举，模拟注解中添加枚举属性
 * 
 * @author jiqinlin
 *
 */
 public enum Gender {
    MAN{
        public String getName(){return "男";}
    },
    WOMEN{
        public String getName(){return "女";}
    }; //记得有“;”
     public abstract String getName();
}
```

注解类MetaAnnotation，模拟注解中添加注解属性  
```java
package com.ljq.test;

 /**
 * 定义一个注解，模拟注解中添加注解属性
 * 
 * @author jiqinlin
 *
 */
 public @interface MetaAnnotation {
    String birthday();
}
```