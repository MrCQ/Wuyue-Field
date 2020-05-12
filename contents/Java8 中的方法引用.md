# Java8 中的方法引用

Java 8中方法也是一种对象，可以By名字来引用。不过**方法引用的唯一用途是支持Lambda的简写**，使用方法名称来表示Lambda

操作符 `::` ，这个操作符把方法引用分成两边，左边是类名或者某个对象的引用，右边是方法名或者是“new”（构造器引用时用到）

 * 对象引用::实例方法名
 * 类名::静态方法名
 * 类名::实例方法名

如何来区分这些使用呢

* 静态方法引用 :	 `ContainingClass::staticMethodName`
* 构造方法引用 :	 `ClassName::new	`
* 类成员方法引用:	 `ContainingType::methodName`
* 实例对象方法引用:  `containingObject::instanceMethodName`

方法引用是在 lamda 基础上的语法糖

```Java
@FunctionInterface
public interface Test<T,R> {
    R test(T t);
}
```

如上所示，我们定义了函数接口（用于lamda表征，只含有一个未实现函数），通过lamda可以实现函数传递，可以采用lamda方式来表示Test，如：

```Java
t -> 2*t 所表示的就是（等价于以下）：
class ClassName implements Test<T, R> {
    R test(T t) {
        return 2 * t;
    }
}
//实际使用过程中会自动解析类型，判断T和R
```

通常匿名类采用 lamda 方式去实现，那么对于完整的类，如何采用更加便利的方式来显示方法传递呢？

对于接收方法引用作为参数的函数，我们可以传递lamda作为参数，但是**怎么能够将已经实现的方法结构传递过去呢**？

我们找到方法引用于lamda表达式之间的对应关系：

* `s -> Car.func(s)     ==   Car::func`   func()是Car的静态方法
* `o -> o.func1()       ==   Car::func1`  func1()是Car的成员方法
* `s -> obj.func2(s)    ==   obj::func2`  obj是Car的实例，func2是Car的方法

那么如何来表示这些函数引用呢？

有几种类型：

* Consumer<T>           方法仅有一个参数，且无返回值
* Predicate<T>          方法仅有一个参数，且返回值是bool类型
* Function<T, R>        方法仅有一个参数，且有返回值, 返回值类型是R
* BiConsumer<T, R>      方法有两个参数，无返回值

究竟用哪个引用类型来表征比如Car::func ， 那就得看func的具体实现结构，尤其是返回值

通常，方法引用会配合流处理的形似进行运用

```java
//以blade-mvc 中的一段代码为例：
blade.scanPackages().stream()
                .flatMap(DynamicContext::recursionFindClasses) //flatMap	传入的是Function
                .map(ClassInfo::getClazz)	//map	传入的是Function
                .filter(ReflectKit::isNormalClass)	//filter	传入的是Predict
                .forEach(this::parseCls);	//forEach	传入的是Consumer

//DynamicContext类中
public static Stream<ClassInfo>recursionFindClasses(String packageName) {
        Scanner        scanner    = Scanner.builder().packageName(packageName).recursive(true).build();
        Set<ClassInfo> classInfos = getClassReader(packageName).readClasses(scanner);
        return classInfos.stream();
    }

//ClassInfo类中
public Class<?> getClazz() {
        return clazz;
    }

//ReflectKit类中
public static boolean isNormalClass(Class<?> cls) {
        return !cls.isInterface() && !Modifier.isAbstract(cls.getModifiers());
    }

//

```

参考：

* [Java 8 Method References explained in 5 minutes](https://blog.idrsolutions.com/2015/02/java-8-method-references-explained-5-minutes/)
* [Java 8 Method Reference: How to Use it](https://www.codementor.io/eh3rrera/using-java-8-method-reference-du10866vx)
* [How to invoke parameterized method with method reference](https://stackoverflow.com/questions/23023618/how-to-invoke-parameterized-method-with-method-reference/23025159#23025159)
