#### Java 虚拟机参数

-Xms 设置初始堆大小

-Xmx 设置最大堆大小

-XX:ReservedCodeCacheSize=240m 这个参数主要用来设置codecache大小，比如我们jit编译的代码都是放在codecache里的，所以codecache如果满了的话，那带来的问题就是无法再jit编译了，而且还会去优化。 因此大家可能碰到这样的问题：cpu一直高，然后发现是编译线程一直高（系统运行到一定时期），这个很大可能是codecache满了，一直去做优化。  

-XX:+PrintGCDetails 打印GC日志

-Verbose:gc 用于垃圾收集时的信息打印 	

-XX:+PrintGCDateStamps  打印GC时间戳

-Xloggc:C:\Users\ligj\Downloads\gc.log GC 把GC日志输出的地方

#### 虚拟机默认参数

在命令行输入一下内容直接查看：

```
java -XX:+PrintFlagsInitail
```

#### 查看虚拟机使用的虚拟机

```
public static void main(String[] args){
        List<GarbageCollectorMXBean> l = ManagementFactory.getGarbageCollectorMXBeans();
        l.forEach(b -> System.out.println(b.getName()));
    }
```

