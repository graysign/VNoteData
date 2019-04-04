客户类和工厂类分开。消费者任何时候需要某种产品，只需向工厂请求即可。消费者无须修改就可以接纳新产品。缺点是当产品修改时，工厂类也要做相应的修改。  
例子：追MM少不了请吃饭了，麦当劳的鸡翅和肯德基的鸡翅都是MM爱吃的东西，虽然口味有所不同，但不管你带MM去麦当劳或肯德基，只管向服务员说“来四个鸡翅”就行了。  
麦当劳和肯德基就是生产鸡翅的Factory  

```java
interface Sample {
    public void say();
}


class SampleA implements Sample {
    public void say() {
        System.out.println("SampleA");
    }
}


class SampleB implements Sample {
    public void say() {
        System.out.println("SampleB");
    }
}


class Factory {
    public static Sample create(int which) {
        if (which == 1) {
            return new SampleA();
        } else if (which == 2) {
            return new SampleB();
        } else {
            return null;
        }
    }
}


public class Test {
    public static void main(String[] args) {
        Sample test1 = Factory.create(1);
        test1.say();

        Sample test2 = Factory.create(2);
        test2.say();
    }
}

```