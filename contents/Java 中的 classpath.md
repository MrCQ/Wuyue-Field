# Java 中的 classpath

## classpath 是什么

编写 java 程序时，每当依赖其他一些类实现时，通常会以 import 方式导入依赖类，那么实际上这个类真正的class文件究竟在什么位置呢？

可以确定的是，类的前缀包路径为该类文件的存储路径，但此路径仅仅为相对路径，相对的依据是什么，自然就是 classpath

classpath 就是告诉JVM去哪里加载所需要的类文件

## classpath 下有什么

通常来说，classpath下包含两类：

* 项目的编译后类文件路径（package的顶层路径），例如IDEA中项目的输出路径`path/path1/target/output`
* Jar 文件路径（第三方依赖）

## 如何指定classpath内容

在系统安装JDK时候已经对classpath进行了如下设置：

```
export JAVA_HOME=/usr/lib/jvm/jdk1.8.0_144  
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export PATH=${JAVA_HOME}/bin:$PATH  
```

或者运行时重新设置：

`java -cp "/path:/path1"`

注意：以上方式会覆盖原有classpath的设置，所以在混用情况下`/xxx/xx.jar:$CLASSPATH`

如何查看当前项目的classpath呢：

```
System.out.println(System.getProperty("java.class.path"))
```

## classpath 文件顺序

例如当classpath中加入a.jar与b.jar，谁前谁后放置对于系统运行有影响吗？

有影响！

若a.jar在前，当a.jar 与 b.jar 包含同名类时，同一加载器按序加载是从a.jar 记载到该类，那么会忽略后续加载到的该同名类，顺序加载！

那么在classpath中，jdk中类库顺序在前，当前项目次之，项目依赖类库顺序再次，如果依赖库通过maven管理，那么顺序等同于maven中的声明顺序。

## classpath 由谁来加载

classpath下的类有系统加载器加载

