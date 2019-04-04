把一个类的接口变换成客户端所期待的另一种接口，从而使原本因接口原因不匹配而无法一起工作的两个类能够一起工作。适配类可以根据参数返回一个合适的实例给客户端。
```java
interface ICar {
    publicvoid startCar();

    publicvoid stopCar();

    publicvoid fixCar();
}

class CarAdapter implements ICar {
    publicvoid fixCar() {
    }

    publicvoid startCar() {
    }

    publicvoid stopCar() {
    }
}

class Car extends CarAdapter {
    publicvoid fixCar() {
        System.out.println("破车，又坏了，还要去修车");
    }
}


public class Test {
    public static void main(String[] args) {
        ICar car = new Car();
        car.fixCar();
    }
}

```