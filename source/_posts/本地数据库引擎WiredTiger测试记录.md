---
title: 本地数据库引擎WiredTiger测试记录
date: 2019-07-29 18:45:11
tags:
	- 数据库
---

项目上有一个性能优化的需求，需要进一步优化远程调用Redis导致的RT偏高问题，目前的想法是做两级缓存，通过加入一个基于嵌入式数据库引擎的本地缓存，来降低对远端redis的依赖；大致的结构如下：

业务服务 ---> 本地数据库引擎 -> 远端redis

之所以用数据库引擎，是因为在满足QPS的情况下，想尽可能本地化更多的数据，直接用内存太奢侈了。

对于读操作：优先读取本地，本地没有的话，再读区远端redis，远端的值会写回到本地

对于写操作：直接写本地，然后后台线程异步同步至远端，其中业务逻辑可以容忍短时的数据不一致情况。

WiredTiger是mongodb收购过来的一个存储引擎，从2.3版本之后，便是mongodb的默认引擎。

除了WiredTiger之外，一些本地的嵌入式数据库引擎还有：[地址](http://embedded-databases.com/)

相比Google的LevelDb引擎的性能对比：[链接](https://github.com/wiredtiger/wiredtiger/wiki/LevelDB-Benchmark)

本文主要记录WiredTiger安装的流程，以及一个简单的Java操作示例，安装的环境：

```
$system_profiler SPSoftwareDataType
Software:

    System Software Overview:

      System Version: macOS 10.14.6 (18G48f)
      Kernel Version: Darwin 18.7.0
      Boot Volume: Untitled
      Boot Mode: Normal
      Computer Name: yuanke’s MacBook Ali
      User Name: yuanke wei (yuankewei)
      Secure Virtual Memory: Enabled
      System Integrity Protection: Enabled
      Time since boot: 17 days 5:20
```

（1）git下载相关的代码，并切换到稳定分支

```
$git clone git://github.com/wiredtiger/wiredtiger.git

$cd wiredtiger/

$git checkout -b mongodb-4.4 origin/mongodb-4.4 
```

（2）安装相关编译工具

```
#采用brew安装
$brew install autoconf automake libtool swig
```

（3）开始编译

```
$./autogen.sh

#mac下查看JAVA_HOME的命令为：
$/usr/libexec/java_home

#**编辑生成的configure文件，
17620行开始，替换

case "$host_os" in
        darwin*)        _JTOPDIR=`echo "$_JTOPDIR" | sed -e 's:/[^/]*$::'`
                        _JINC="$_JTOPDIR/Headers";;
        *)              _JINC="$_JTOPDIR/include";;
esac

为

case "$host_os" in
        *)              _JINC="$_JTOPDIR/include";;
esac

$./configure --enable-java --enable-python

#开始编译
$make

#安装相关库文件到/usr/local/
$make install

```

（4）采用idea建立一个maven工程

```
#wiredtiger安装之后，java的jar包和jni本地库的位置为：
$ls /usr/local/share/java/wiredtiger-3.2.0/
libwiredtiger_java.0.dylib libwiredtiger_java.dylib   wiredtiger.jar
libwiredtiger_java.a       libwiredtiger_java.la

编辑idea的工程，引入对应的jar包
Project Settings -> Libraries

另外，运行测试程序时，还需要加入jni本地库的路径，idea的jvm运行参数重加入：
-Djava.library.path=/usr/local/share/java/wiredtiger-3.2.0

```

完整的测试程序如下

```
import com.wiredtiger.db.*;

public class Main {
    public static void main(String[] args) {
        Connection conn;
        Session s;
        Cursor c;

        try {
            //保证目录是存在的/Users/yuankewei/temp/WT_HOME
            conn = wiredtiger.open("/Users/yuankewei/temp/WT_HOME", "create");
            s = conn.open_session(null);
        } catch (WiredTigerException wte) {
            System.err.println("WiredTigerException: " + wte);
            return;
        }

        /*! [access example connection] */
        try {
            /*! [access example table create] */
            //创建了一个table，key是一个String，Value也是一个String
            s.create("table:t", "key_format=S,value_format=S");
            /*! [access example table create] */
            /*! [access example cursor open] */
            c = s.open_cursor("table:t", null, null);
            /*! [access example cursor open] */
        } catch (WiredTigerException wte) {
            System.err.println("WiredTigerException: " + wte);
            return;
        }

        /*! [access example cursor insert] */
        //插入一个数据
        try {
            c.putKeyString("foo");
            c.putValueString("bar");
            c.insert();
        } catch (WiredTigerPackingException wtpe) {
            System.err.println("WiredTigerPackingException: " + wtpe);
        } catch (WiredTigerException wte) {
            System.err.println("WiredTigerException: " + wte);
        }

        /*! [access example cursor insert] */
        //再插入一行数据
        try {
            c.putKeyString("foo1");
            c.putValueByteArray("bar1".getBytes());
            c.insert();
        } catch (WiredTigerPackingException wtpe) {
            System.err.println("WiredTigerPackingException: " + wtpe);
        } catch (WiredTigerException wte) {
            System.err.println("WiredTigerException: " + wte);
        }

        //根据key查询value
        try {
            c.reset();
            c.putKeyString("foo1");
            c.search();
            System.out.println(c.getValueString());
        } catch (Exception e) {
        }



        /*! [access example cursor insert] */
        /*! [access example cursor list] */
        //遍历数据
        try {
            c.reset();
            while (c.next() == 0) {
                System.out.println("Got: " + c.getKeyString() + ",   " + new String(c.getValueByteArray()));

            }
        } catch (WiredTigerPackingException wtpe) {
            System.err.println("WiredTigerPackingException: " + wtpe);
        } catch (WiredTigerException wte) {
            System.err.println("WiredTigerException: " + wte);
        }
    }
}
```

程序的数据为：

```
bar1
Got: foo,   bar
Got: foo1,   bar1
```
