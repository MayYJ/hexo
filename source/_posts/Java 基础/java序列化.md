---
title: java序列化
date: 2018-05-12 22:46:12
tags: 序列化
categories: java
---

#### 为什么要序列化
当我们创建一个对象后，如果程序终止那么对象就会销毁，但是存在我们需要对对象进行持久化的需求，以便在将来我们取出对象
进行再次利用。序列化就是把对象变成字节码序列来实现轻量级持久化，也方便我们对其在网络上进行传输。
#### JDK 序列化的三种方式

#####  直接方式

```java
public class Person implements Serializable {
    private long serialVersionUID = 1L;
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

public class TestSerializiable {

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream ops = new ObjectOutputStream(bos);
        Person person = new Person("may", 10);
        ops.writeObject(person);

        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bos.toByteArray());
        ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
        Person person1 = (Person) objectInputStream.readObject();
    }
}
```

##### 实现Externalizable接口方式

这种方式需要注意一个问题，必须含有空构造器，因为该方式下是通过空构造器建立对象，然后通过反射设置值

```java
public class Person implements Externalizable {
    private long serialVersionUID = 1L;
    private String name;
    private int age;

    public Person() {}

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(age);
        out.writeObject(name);
    }

    @Override
    public String toString() {
        return "Person{" +
                "serialVersionUID=" + serialVersionUID +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        age = in.readInt();
        name = (String) in.readObject();

    }
}
```

##### 实现私有方法的方式

```java
public class Person implements Serializable {
    private long serialVersionUID = 1L;
    private String name;
    private int age;


    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    private void writeObject(ObjectOutputStream stream) throws IOException {
        stream.writeInt(age);
        stream.writeObject(name);
    }

    @Override
    public String toString() {
        return "Person{" +
                "serialVersionUID=" + serialVersionUID +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }

    private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException {
        age = stream.readInt();
        name = (String) stream.readObject();
    }
}
```

#### writeObject 算法

大致有下面四个步骤：

1. 算法会将关联实例的类的元数据写入文件中

2. 算法会递归的将父类写入文件，直到遇到Object为止

3. 类的元数据写完之后，算法便从最顶端的父类开始写入类实例的真实数据。

4. 算法将递归地将类实例的数据写入文件。

   ![](http://core0.staticworld.net/images/idge/imported/article/jvw/2009/05/jtip050709-fig2-100156512-orig.gif)

下面给一个例子

```java

class parent implements Serializable {
    int parentVersion = 10;
}
 
class contain implements Serializable{
    int containVersion = 11;
}
public class SerialTest extends parent implements Serializable {
    int version = 66;
    contain con = new contain();
 
    public int getVersion() {
        return version;
    }
    public static void main(String args[]) throws IOException {
        FileOutputStream fos = new FileOutputStream("temp.out");
        ObjectOutputStream oos = new ObjectOutputStream(fos);
        SerialTest st = new SerialTest();
        oos.writeObject(st);
        oos.flush();
        oos.close();
    }

```

```txt
AC ED 00 05 73 72 00 0A 53 65 72 69 61 6C 54 65
73 74 05 52 81 5A AC 66 02 F6 02 00 02 49 00 07
76 65 72 73 69 6F 6E 4C 00 03 63 6F 6E 74 00 09
4C 63 6F 6E 74 61 69 6E 3B 78 72 00 06 70 61 72
65 6E 74 0E DB D2 BD 85 EE 63 7A 02 00 01 49 00
0D 70 61 72 65 6E 74 56 65 72 73 69 6F 6E 78 70
00 00 00 0A 00 00 00 42 73 72 00 07 63 6F 6E 74
61 69 6E FC BB E6 0E FB CB 60 C7 02 00 01 49 00
0E 63 6F 6E 74 61 69 6E 56 65 72 73 69 6F 6E 78
70 00 00 00 0B
```

现在我们将逐行解读序列化对象存储在文件中的内容：

- ​    AC ED：STREAM_MAGIC.    序列化协议标识
- ​    00 05：STREAM_VERSION.    流版本号
- ​    0x73：TC_OBJECT.    表示这是一个新对象   

第一步算法将描述类的实例，本例中是对象SerialTest。

- ​    0x72: TC_CLASSDESC.    表示一个新对象
- ​    00 0A: 类名长度
- ​    53 65 72 69 61 6c 54 65 73 74:SerialTest（对应ASC码16进制），类名
- ​    05 52 81 5A AC 66 02 F6: 序列化版本号
- ​    0x02: 表示该对象支持序列化
- ​    00 02:  表示类中域的个数（本例中SerialTest类有两个域version和con）

接下来将会写域int version = 66;。

- ​    0x49:  域类型码，49表示"I"，代表int类型
- ​    00 07: 域名长度
- ​    76 65 72 73 69 6F 6E: version，域的名称

接下来算法将会写域contain con = new contain();。由于con是一个对象，只会写入规范的JVM对象摘要。

- ​    0x74: TC_STRING.    表示一个新字符串
- ​    00 09: 表示字符串长度
- ​    4C 63 6F 6E 74 61 69 6E 3B: Lcontain;，标准JVM摘要
- ​    0x78: TC_ENDBLOCKDATA，表示该对象可选的数据块末端

下一步算法会将SerialTest类的父类parent写入文件。

- ​    0x72: TC_CLASSDESC.    表示一个新对象
- ​    00 06: 类名长度
- ​    70 61 72 65 6E 74:  parent，类名
- ​    0E DB D2 BD 85 EE 63 7A:  序列化版本号
- ​    0x02: 表示该对象支持序列化
- ​    00 01:  表示类中域的个数（本例中parent类有一个域parentVersion）

下一步算法将parent类的域写入。parent类只有一个域int parentVersion = 100;。

- ​    0x49:  域类型码，49表示"I"，代表int类型
- ​    00 0D: 域名长度
- ​    70 61 72 65 6E 74 56 65 72 73 69 6F 6E: parentVersion，域的名称
- ​    0x78: TC_ENDBLOCKDATA，表示该对象可选的数据块末端
- ​    0x70: TC_NULL，表示向上已经没有父类

目前为止，算法已经将类实例以及其父类的描述信息写入文件，接下来要将实例对应的真是变量值写入文件，写入顺序是从最顶层父类开始的。

- ​    00 00 00 0A: 10，父类实例变量parentVersion的值
- ​    00 00 00 42: 66，类SerialTest中实例变量version的值

接下来的数据比较有趣，算法将实例变量con对应的类信息写入了文件

- ​    0x73：TC_OBJECT.    表示这是一个新对象
- ​    0x72: TC_CLASSDESC.    表示一个新对象
- ​    00 07: 类名长度
- ​    63 6F 6E 74 61 69 6E: contain，类名
- ​    FC BB E6 0E FB CB 60 C7: 序列化版本号
- ​    0x02: 表示该对象支持序列化
- ​    00 01:  表示类中域的个数

接下来算法将会写类中的域int containVersion = 11;。

- ​    0x49:  域类型码，49表示"I"，代表int类型
- ​    00 0E: 域名长度
- ​    63 6F 6E 74 61 69 6E 56 65 72 73 69 6F 6E: containVersion，域名
- ​    0x78: TC_ENDBLOCKDATA，表示该对象可选的数据块末端

最后算法将会检查类是否存在父类，如果有父类，则继续写父类的信息；如果没有父类，则写入TC_BULL。

- ​    0x70: TC_NULL，表示向上已经没有父类

最后算法将写入类的实例变量对应的数值。

- ​    00 00 00 0B: 11，类Contain中实例变量containVersion的值

#### readObject 方法源码解析

```java
 private Object readObject0(boolean unshared) throws IOException {
        ...
        byte tc;
        while ((tc = bin.peekByte()) == TC_RESET) {
            bin.readByte();
            handleReset();
        }

        depth++;
        totalObjectRefs++;
     // 从上面序列化的算法，我们可以知道每一种类型都对应一个byte的标志，下面就是根据上面读取到的这个标志去选择相应的解析方法
        try {
            switch (tc) {
                case TC_NULL:
                    return readNull();

                case TC_REFERENCE:
                    return readHandle(unshared);

                case TC_CLASS:
                    return readClass(unshared);

                case TC_CLASSDESC:
                case TC_PROXYCLASSDESC:
                    return readClassDesc(unshared);

                case TC_STRING:
                case TC_LONGSTRING:
                    return checkResolve(readString(unshared));

                case TC_ARRAY:
                    return checkResolve(readArray(unshared));

                case TC_ENUM:
                    return checkResolve(readEnum(unshared));

                case TC_OBJECT:
                    return checkResolve(readOrdinaryObject(unshared));

                case TC_EXCEPTION:
                    IOException ex = readFatalException();
                    throw new WriteAbortedException("writing aborted", ex);

                case TC_BLOCKDATA:
                case TC_BLOCKDATALONG:
                    if (oldMode) {
                        bin.setBlockDataMode(true);
                        bin.peek();             // force header read
                        throw new OptionalDataException(
                            bin.currentBlockRemaining());
                    } else {
                        throw new StreamCorruptedException(
                            "unexpected block data");
                    }

                case TC_ENDBLOCKDATA:
                    if (oldMode) {
                        throw new OptionalDataException(true);
                    } else {
                        throw new StreamCorruptedException(
                            "unexpected end of block data");
                    }

                default:
                    throw new StreamCorruptedException(
                        String.format("invalid type code: %02X", tc));
            }
        } finally {
            depth--;
            bin.setBlockDataMode(oldMode);
        }
    }

```

从readOrdinaryObject 作为入口区分析

```java
    private Object readOrdinaryObject(boolean unshared)
        throws IOException
    {
        if (bin.readByte() != TC_OBJECT) {
            throw new InternalError();
        }

        // 读取到所有的元数据
        ObjectStreamClass desc = readClassDesc(false);
        desc.checkDeserialize();

        Class<?> cl = desc.forClass();
        if (cl == String.class || cl == Class.class
                || cl == ObjectStreamClass.class) {
            throw new InvalidClassException("invalid class descriptor");
        }

        Object obj;
        try {
        // 创建了对象
            obj = desc.isInstantiable() ? desc.newInstance() : null;
        } catch (Exception ex) {
            throw (IOException) new InvalidClassException(
                desc.forClass().getName(),
                "unable to create instance").initCause(ex);
        }

        passHandle = handles.assign(unshared ? unsharedMarker : obj);
        ClassNotFoundException resolveEx = desc.getResolveException();
        if (resolveEx != null) {
            handles.markException(passHandle, resolveEx);
        }
        // 选择序列化的方式
        if (desc.isExternaliable()) {
            readExternalData((Externalizable) obj, desc);
        } else {
            readSerialData(obj, desc);
        }

        handles.finish(passHandle);

        // 有readResolve 方法那么调用其
        if (obj != null &&
            handles.lookupException(passHandle) == null &&
            desc.hasReadResolveMethod())
        {
            Object rep = desc.invokeReadResolve(obj);
            if (unshared && rep.getClass().isArray()) {
                rep = cloneArray(rep);
            }
            if (rep != obj) {
                // Filter the replacement object
                if (rep != null) {
                    if (rep.getClass().isArray()) {
                        filterCheck(rep.getClass(), Array.getLength(rep));
                    } else {
                        filterCheck(rep.getClass(), -1);
                    }
                }
                handles.setObject(passHandle, obj = rep);
            }
        }

        return obj;
    }

```

#### 序列版本号serialVersionUID作用

用于控制类的版本，如果序列化数据中的serialVersionUID 与本地类里面的serialVersionUID不一样那么，不允许反序列化

#### 一些应该注意的问题

1. 可序列化的子类，那么其所有父类都是可序列化的
2. 当一个对象被序列化时，只序列化对象的非静态成员变量，不能序列化任何成员方法和静态成员变量。
3. 如果一个可序列化的对象包含对某个不可序列化的对象的引用，那么整个序列化操作将会失败，并且会抛出一个NotSerializableException。可以通过将这个引用标记为transient，那么对象仍然可以序列化。对于一些比较敏感的不想序列化的数据，也可以采用该标识进行修饰。





