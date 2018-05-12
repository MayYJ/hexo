---
title: java序列化
date: 2018-05-12 22:46:12
tags: 序列化
categories: java
---

#### 为什么要序列化
当我们创建一个对象后，如果程序终止那么对象就会销毁，但是存在我们需要对对象进行持久化的需求，以便在将来我们取出对象
进行再次利用。序列化就是把对象变成字节码序列来实现轻量级持久化，也方便我们对其在网络上进行传输。
#### 怎么序列化
```
public class Stu implements Serializable{
    private String name;
    private int age;
    private transient String password;

    public Stu(String name, int age, String password) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getName() {
        return name;
    }

}

public class TestSerializable {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        Stu stu = new Stu("May",20, "123456");
        ObjectOutputStream ops = new ObjectOutputStream(new FileOutputStream("/home/may/Documents/temp.txt"));
        ops.writeObject(stu);
        ops.close();
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("/home/may/Documents/temp.txt"));
        Stu stu1 = (Stu) ois.readObject();
        System.out.println(stu1.getPassword());
    }
}
```
以上便是简单的实现对象的序列化，只需要在类上添加Serializable标记接口，然后就可以直接使用ObjectOutputStream和ObjecInputStream对对象进行序列化和反序列化。
#### transient关键字
在序列化的时候，有些字段我们不想默认序列化，比如说用户的密码等；这个时候我们就可以使用这个关键字，对标记的字段屏蔽默认序列化。
#### 寻找类
在我们进行反序列化的时候，被序列化对象的java文件应该在同一个目录下，而且版本相同即在序列化后没有进过修改，不然在反序列化的时候会抛出ClassNotFoundException。
#### 序列化的控制
有些时候我们不想按照默认的序列化方式进行，我们想定义自己对一个对象的序列化的方式，当然上面transient是一个方式。我们还可以使用Externalizable接口，重载writeExternal和readExternal方法。

```
public class Stu implements Externalizable{
    private String name;
    private int age;
    private transient String password;

    public Stu(String name, int age, String password) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getName() {
        return name;
    }

    public void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();
    }

    public void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();
    }
}
```
这里的defaultReadObject方法就是会采用默认的对象的序列化方式。

#### Externalizable的替代方案
直接在实现了Serialization类里面实现下面两个方法
```
private void WriterObject()

private void readObject()
```
在序列化和反序列化的时候在调用ObjectOutputStream.writeObject()和ObjectInputStream.readObject()的时候会检查object是不是有自己的writeObject和readObject方法。