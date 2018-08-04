---
title: String源码解析
date: 2018-03-12 19:19:44
tags: String对象
categories: java
---

####变化

在JDK 8之前，String的源码实现都是通过使用char数组接收字符串，但是每个char字符是由两个字符组成，因为java内部使用UTF-16实现；如果一个字符串含有英文字符，那么这些英文字符的前 8 比特都将为 0，因为一个ASCII字符都能被单个字节来表示。然而我们在使用字符串的时候只需8bit的情况占大多数。由于这种情况jvm堆空间通常很大一部分被字符串占据。

在java1.6时候引入了一个新的虚拟机参数 UseCompressedStrings；字符串将以byte数组的形式存储，代替原来的char。然而在JDK 7中被移除，主要原因在于它将带来一些如法预料的性能问题。

#### JDK 9中String对象的实现

#####实现方式

```
private final byte[] value;
```

用于存储String的字节编码

```
private final byte coder;
static	 final boolean COMPACT_STRINGS = true;
static final byte LATIN1 = 0;
static final byte UTF16 = 1;
```

这些是JDK 9用于解决字符串占用内存空间大的方案的变量

每个String对象都拥有一个coder变量，它有两个值:0或者1,分别对应LATIN1和UTF16；当coder=0时表示这个String对象是用LATIN1字符集进行编码，当coder=1时表示String对象是用UTF16进行编码。	

我们可以根据构造函数作为切入点进行探究

```
    public String(char[] value) {
        this((char[])value, 0, value.length, (Void)null);
    }
    
    String(char[] value, int off, int len, Void sig) {
        if (len == 0) {
            this.value = "".value;
            this.coder = "".coder;
        } else {
            if (COMPACT_STRINGS) {
                byte[] val = StringUTF16.compress(value, off, len);
                if (val != null) {
                    this.value = val;
                    this.coder = 0;
                    return;
                }
            }

            this.coder = 1;
            this.value = StringUTF16.toBytes(value, off, len);
        }
    }
```

  从第二个构造函数我们可以很清楚的看到实现方式：

```java
    if (COMPACT_STRINGS) {
          byte[] val = StringUTF16.compress(value, off, len);
          if (val != null) {
              this.value = val;
              this.coder = 0;
              return;
            }
        }
          this.coder = 1;
          this.value = StringUTF16.toBytes(value, off, len);
        
    //StringUTF16的方法    
    public static byte[] compress(char[] val, int off, int len) {
        byte[] ret = new byte[len];
        return compress((char[])val, off, ret, 0, len) == len ? ret : null;
    }
       
    //StringUTF16的方法
    public static int compress(char[] src, int srcOff, byte[] dst, 
                               int dstOff, int len）{
        for(int i = 0; i < len; ++i) {
            char c = src[srcOff];
            if (c > 255) {
                len = 0;
                break;
            }

            dst[dstOff] = (byte)c;
            ++srcOff;
            ++dstOff;
        }

        return len;
    }
 
```

首先根据String的类变量COMPACT_STRINGS 默认为true判断是否对String对象进行压缩；根据compress()方法我们能够看出来 压缩是根据字符串是不是全部能够根据ASCII码表找对应的编码，如果有一个字符不符合，那么就会返回0，从而不进行压缩，而是直接采用UTF16进行编码。



---

##### 内置的比较器

```
    public static final Comparator<String> CASE_INSENSITIVE_ORDER = new String.CaseInsensitiveComparator();
```

这个是String内部默认的排序方式

```
    private static class CaseInsensitiveComparator implements Comparator<String>, Serializable {
        private static final long serialVersionUID = 8575799808933029326L;

        private CaseInsensitiveComparator() {
        }

        public int compare(String s1, String s2) {
            byte[] v1 = s1.value;
            byte[] v2 = s2.value;
            if (s1.coder() == s2.coder()) {
                return s1.isLatin1() ? StringLatin1.compareToCI(v1, v2) : StringUTF16.compareToCI(v1, v2);
            } else {
                return s1.isLatin1() ? StringLatin1.compareToCI_UTF16(v1, v2) : StringUTF16.compareToCI_Latin1(v1, v2);
            }
        }

        private Object readResolve() {
            return String.CASE_INSENSITIVE_ORDER;
        }
    }
```

从代码也可以很容易看出来就是从第一个字符开始挨着挨着的比较，如果有一个字符不一样，那么就会直接比较这两个字符的大小，从而得出两个字符串的大小。

##### 从源码可以学到的东西

1. 检查方法参数是否正确

   ```java
       public String(char[] value, int offset, int count) {
           this(value, offset, count, rangeCheck(value, offset, count));
       }
       
        String(char[] value, int off, int len, Void sig) {
           if (len == 0) {
               this.value = "".value;
               this.coder = "".coder;
           } else {
               if (COMPACT_STRINGS) {
                   byte[] val = StringUTF16.compress(value, off, len);
                   if (val != null) {
                       this.value = val;
                       this.coder = 0;
                       return;
                   }
               }

               this.coder = 1;
               this.value = StringUTF16.toBytes(value, off, len);
           }
       }

   ```

   我们可以用Void类型作为参数，接受如果运行正确的方法返回值为void的方法。

   2. String类中存在的同步

      ```
          public boolean contentEquals(CharSequence cs) {
              if (cs instanceof AbstractStringBuilder) {
                  if (cs instanceof StringBuffer) {
                      synchronized(cs) {
                          return this.nonSyncContentEquals((AbstractStringBuilder)cs);
                      }
                  } else {
                      return this.nonSyncContentEquals((AbstractStringBuilder)cs);
                  }
              } else if (cs instanceof String) {
                  return this.equals(cs);
              } else {
                  int n = cs.length();
                  if (n != this.length()) {
                      return false;
                  } else {
                      byte[] val = this.value;
                      if (this.isLatin1()) {
                          for(int i = 0; i < n; ++i) {
                              if ((val[i] & 255) != cs.charAt(i)) {
                                  return false;
                              }
                          }
                      } else if (!StringUTF16.contentEquals(val, cs, n)) {
                          return false;
                      }

                      return true;
                  }
              }
          }
          
          private boolean nonSyncContentEquals(AbstractStringBuilder sb) {
              int len = this.length();
              if (len != sb.length()) {
                  return false;
              } else {
                  byte[] v1 = this.value;
                  byte[] v2 = sb.getValue();
                  if (this.coder() == sb.getCoder()) {
                      int n = v1.length;

                      for(int i = 0; i < n; ++i) {
                          if (v1[i] != v2[i]) {
                              return false;
                          }
                      }

                      return true;
                  } else {
                      return !this.isLatin1() ? false : StringUTF16.contentEquals(v1, v2, len);
                  }
              }
          }
      ```

      在多线程中可能存在在比较的过程中，被比较的字符串被其它县城关改变，造成运行出错。