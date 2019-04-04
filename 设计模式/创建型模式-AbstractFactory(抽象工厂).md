 核心工厂类不再负责所有产品的创建，而是将具体创建的工作交给子类去做，成为一个抽象工厂角色，仅负责给出具体工厂类必须实现的接口，而不接触哪一个产品类应当被实例化这种细节。  
例子：请MM去麦当劳吃汉堡，不同的MM有不同的口味，要每个都记住是一件烦人的事情，我一般采用Factory Method模式，带着MM到服务员那儿，说“要一个汉堡”，具体要什么样的  
汉堡呢，让MM直接跟服务员说就行了。

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


abstract class Factory {
    public abstract Sample create();
}


class FactoryA extends Factory {
    public Sample create() {
        return new SampleA();
    }
}


class FactoryB extends Factory {
    public Sample create() {
        return new SampleB();
    }
}


public class Test {
    public static void main(String[] args) {
        Factory factoryA = new FactoryA();
        Sample test1 = factoryA.create();
        test1.say();

        Factory factoryB = new FactoryB();
        Sample test2 = factoryB.create();
        test2.say();
    }
}

```