将产品的内部表象和产品的生成过程分割开来，从而使一个建造过程生成具有不同的内部表象的产品对象。建造模式使得产品内部表象可以独立的变化，客户不必知道产品内部组成的细节。  
建造模式可以强制实行一种分步骤进行得建造过程。  

```java
class House {
    private String base;
    private String wall;
    private String roof;

    public String getBase() {
        return base;
    }

    public void setBase(String base) {
        this.base = base;
    }

    public String getWall() {
        return wall;
    }

    public void setWall(String wall) {
        this.wall = wall;
    }

    public String getRoof() {
        return roof;
    }

    public void setRoof(String roof) {
        this.roof = roof;
    }

    public String toString() {
        return this.base + " " + this.wall + " " + this.roof;
    }
}


interface Builder {
    public void bulidPartA();

    public void buildPartB();

    public void buildPartC();

    public House getRusult();
}


class HouseBuilder implements Builder {
    private House house;

    public HouseBuilder() {
        house = new House();
    }

    public void bulidPartA() {
        house.setBase("地基建造完成");
    }

    public void buildPartB() {
        house.setWall("墙建造完成");
    }

    public void buildPartC() {
        house.setRoof("屋顶建造完成");
    }

    public House getRusult() {
        return house;
    }
}


class Director {
    public House construct(Builder bulider) {
        bulider.bulidPartA();
        bulider.buildPartB();
        bulider.buildPartC();

        return bulider.getRusult();
    }
}


public class Test {
    public static void main(String[] args) {
        Director d = new Director();
        House house = d.construct(new HouseBuilder());
        System.out.println(house);
    }
}

```

这时候，我们就可以任意建造房屋，可以只造地基，只造墙，只造屋顶，任意组合……（- -那还是房子吗）