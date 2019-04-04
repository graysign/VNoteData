将抽象化与实现化脱耦，使得二者可以独立的变化，也就是说将他们之间的强关联变成弱关联，也就是指在一个软件系统的抽象化和实现化之间使用组合/聚合关系而不是继承关系，
从而使两者可以独立的变化。
例子：比如你要做山寨产品，更新一定要快，什么流行生产什么，但工厂还是那个工厂。

```java
abstract class Product {
    public abstract void beforeProducted();

    public abstract void beforeSelled();
}


class House extends Product {
    @Override
    public void beforeProducted() {
        System.out.println("生产出来房子是这样的");
    }

    @Override
    public void beforeSelled() {
        System.out.println("生产出来的房子卖出去了");
    }
}


class Clothes extends Product {
    @Override
    public void beforeProducted() {
        System.out.println("生产出来衣服是这样的");
    }

    @Override
    public void beforeSelled() {
        System.out.println("生产出来的衣服卖出去了");
    }
}


abstract class Crop {
    private Product product;

    public Crop(Product product) {
        this.product = product;
    }

    public void makeMoney() {
        this.product.beforeProducted();
        this.product.beforeSelled();
    }
}


class shanzaiCrop extends Crop {
    public shanzaiCrop(Product product) {
        super(product);
    }

    @Override
    public void makeMoney() {
        super.makeMoney();
        System.out.println("狂赚钱");
    }
}


public class Test {
    public static void main(String[] args) {
        Crop housecrop = new shanzaiCrop(new House());
        housecrop.makeMoney();

        Crop clothcrop = new shanzaiCrop(new Clothes());
        clothcrop.makeMoney();
    }
}

```