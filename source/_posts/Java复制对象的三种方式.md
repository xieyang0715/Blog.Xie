---

title: Java复制对象的三种方式
date: 2022-05-06 14:31:00
categories: 
   - [Java]
toc: true
---



两个对象如果直接使用=赋值,那么由于对象复制的是对象的地址引用，在赋值和被赋值对象的任一对象改变都会相互影响。这里有三种方式解决这个问题

<!--more-->

##### 重写Cloneable接口的clone方法

在需要克隆的对象继承Cloneable接口，并实现clone方法

```
@Data
public class Dog implements Cloneable {
    @Override
    public Dog clone() throws CloneNotSupportedException {
        return (Dog) super.clone();
    }

    private String category;

    private Integer age;

    public Dog(String category, Integer age) {
        this.category = category;
        this.age = age;
    }
}
```

调用clone方法克隆对象

```
@GetMapping("/hello22")
    public void hello() throws CloneNotSupportedException {
        Dog dog = new Dog("哈士奇", 2);
        Dog dog1 = dog;
        Dog dog2 = dog.clone();

        dog.setCategory("柴犬");

        System.out.println(dog.getCategory());//柴犬
        System.out.println(dog1.getCategory());//柴犬
        System.out.println(dog2.getCategory());//哈士奇
    }
```

如果克隆对象Dog中还有引用对象，则该引用对象需要继承Cloneable接口，并实现clone方法才能实现深度克隆

如Dog引用了Organ，则器官对象Organ也需要继承Cloneable接口，并实现clone方法

    @Data
    public class Dog implements Cloneable {
        @Override
        public Dog clone() throws CloneNotSupportedException {
            return (Dog) super.clone();
        }
        private String category;
    
        private Integer age;
    
        //器官对象
        private Organ organ;
    
        public Dog(String category, Integer age) {
            this.category = category;
            this.age = age;
        }
    }
##### 使用工具类BeanUtils的copyProperties方法

对象

```
@Data
public class Dog {
    private String category;

    private Integer age;

    public Dog() {
    }

    public Dog(String category, Integer age) {
        this.category = category;
        this.age = age;
    }
}
```

调用BeanUtils.copyProperties(source,target)

```
@GetMapping("/hello22")
    public void hello() throws CloneNotSupportedException {
        Dog dog = new Dog("哈士奇", 2);
        Dog dog1 = dog;
        Dog dog2 = new Dog();
        BeanUtils.copyProperties(dog,dog2);

        dog.setCategory("柴犬");

        System.out.println(dog.getCategory());//柴犬
        System.out.println(dog1.getCategory());//柴犬
        System.out.println(dog2.getCategory());//哈士奇
    }
```

##### 用序列化反序列化复制对象

对象继承Serializable

```
@Data
public class Dog implements Serializable {
    private String category;

    private Integer age;

    public Dog() {
    }

    public Dog(String category, Integer age) {
        this.category = category;
        this.age = age;
    }
}
```

实现复制方法

```
public <T extends Serializable>T CopyObject(T source) throws Exception {
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(source);

        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(byteArrayOutputStream.toByteArray());
        ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
        T o = (T)objectInputStream.readObject();
        return o;
    }
```

调用复制

```
 @GetMapping("/hello22")
    public void hello() throws Exception {
        Dog dog = new Dog("哈士奇", 2);
        Dog dog1 = dog;
        Dog dog2 = CopyObject(dog);

        dog.setCategory("柴犬");

        System.out.println(dog.getCategory());//柴犬
        System.out.println(dog1.getCategory());//柴犬
        System.out.println(dog2.getCategory());//哈士奇
    }
```

