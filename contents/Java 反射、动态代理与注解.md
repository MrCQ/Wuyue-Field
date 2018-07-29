# Java 反射、动态代理与注解

本文主要关注与Java的反射、动态代理以及注解三个方面，阐释各自的含义与作用，列出具体使用示例。

## 反射

反射是指程序可以访问，检测，修改它本身状态或行为的一种能力

简单说：反射机制可以让你在程序运行时，拿到任意一个类的属性和方法并调用它。

反射的主要功能：

* 运行时构造一个类的对象；
* 运行时获取一个类所具有的的成员变量和方法；
* 运行时调用任意一个对象的方法；
* 生成动态代理； 其实反射最主要的功能我觉得是与框架搭配使用。

首先需要知道Class这个类，它的全称是java.lang.Class类。java是面向对象的语言，讲究万物皆对象，即使强大到一个类，它依然是另一个类（Class类）的对象，换句话说，普通类是Class类的对象，即Class是所有类的类

但是Class类本身并不能在程序中被初始化，因为Class中所有的构造函数都被定义为private，也就意味着Class只能被jvm初始化

通常，可以通过以下三种方式得到普通类的Class对象

```java
Class c1 = Test.class; //这说明任何一个类都有一个隐含的静态成员变量class，这种方式是通过获取类的静态成员变量class得到的
Class c2 = test.getClass(); //test是Test类的一个对象，这种方式是通过一个类的对象的getClass()方法获得的 
Class c3 = Class.forName("com.catchu.me.reflect.Test"); //这种方法是Class类调用forName方法，通过一个类的全量限定名获得
```

### 反射的一般操作

jdk的java.lang.reflect包中提供了大量与反射相关的数据结构类，将普通类相关的组件都用发射类对象来描述

#### 操作类的构造函数

通过具体的反射类来操作一般类的构造函数，万物皆对象，类的构造函数是`java.lang.reflect.Constructor`类的对象

```java
public Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes) //  获得该类所有的构造器，不包括其父类的构造器
public Constructor<T> getConstructor(Class<?>... parameterTypes) // 获得该类所有public构造器，包括父类
//具体操作如下
Constructor<?>[] allConstructors = class1.getDeclaredConstructors();//获取class对象的所有声明构造函数 
Constructor<?>[] publicConstructors = class1.getConstructors();//获取class对象public构造函数 
Constructor<?> constructor = class1.getDeclaredConstructor(String.class);//获取指定声明构造函数（局部变量是一个字符串类型的） 
Constructor publicConstructor = class1.getConstructor(String.class);//获取指定声明的public构造函数
```

调用：可以通过获取的`Constructor`对象来实现对目标类的初始化，如

```java
constructor.newInstance("name");
//尤其是当构造函数定义为private类型时，不能通过new直接初始化对象，但是利用Constructor依然可以使用
```

#### 操作类的成员变量

万物皆对象，类的成员变量是`java.lang.reflect.Field`类的对象

```java
public Field getDeclaredField(String name) // 获得该类自身声明的所有变量，不包括其父类的变量
public Field getField(String name) // 获得该类自所有的public成员变量，包括其父类变量
//具体实现
Field[] allFields = class1.getDeclaredFields();//获取class对象的所有属性 
Field[] publicFields = class1.getFields();//获取class对象的public属性 
Field ageField = class1.getDeclaredField("age");//获取class指定属性 
Field desField = class1.getField("des");//获取class指定的public属性

注意：
field.setAccessible(true); //如果所操作的成员变量是private类型，需要设置其发射对象field的accessible属性为true，才能进行下一步的操作
```

调用：可以通过成员变量的反射类对象Field直接获取到变量值，如

```java
instace...
Field field = class1.getDeclaredField("field");
field.setAccessible(true) //若field成员变量为private类型
field.get(instance) //获取到instace实例对象的field成员，返回的是Object，需要强转
```

#### 操作类的成员方法

万物皆对象，类的成员方法是`java.lang.reflect.Method`的对象

```java
public Method getDeclaredMethod(String name, Class<?>... parameterTypes) // 得到该类所有的方法，不包括父类的 
public Method getMethod(String name, Class<?>... parameterTypes) // 得到该类所有的public方法，包括父类的
//具体使用:
Method[] methods = class1.getDeclaredMethods();//获取class对象的所有声明方法 
Method[] allMethods = class1.getMethods();//获取class对象的所有public方法 包括父类的方法 
Method method = class1.getMethod("info", String.class);//返回此class1对应的public修饰的方法名是info的，包含一个String类型变量作为参数的方法
Method declaredMethod = class1.getDeclaredMethod("info", String.class);//返回此Class对象对应类的、带指定形参列表的方法
```

调用：可以通过成员方法的反射类对象 Method 对实例对象进行特定方法的运行期调用，如

```java
instance...
Method method = class1.getDeclaredMethod("info",String.class);
method.invoke(instance,"name") //第一个参数是指定运行此方法的实例对象，之后的都是需要传递给成员方法的参数，返回Object
```

#### 操作类或方法的注解

万物皆对象，注解是`java.lang.reflect.Annotation`的对象

```java
Annotation[] annotations = (Annotation[]) class1.getAnnotations();//获取class对象的所有注解 
Annotation annotation = (Annotation) class1.getAnnotation(Deprecated.class);//获取class对象指定注解 
Annotation[] annotations = (Annotation[]) method1.getAnnotations();//获取特定类的指定方法上的所有注解 
Annotation annotation = (Annotation) method1.getAnnotation(Deprecated.class);//获取特定类的指定方法上的特定注解 

Type genericSuperclass = class1.getGenericSuperclass();//获取class对象的直接超类的 
Type Type[] interfaceTypes = class1.getGenericInterfaces();//获取class对象的所有接口的type集合
```

#### 获取Class对象的其他信息
```java
boolean isPrimitive = class1.isPrimitive();//判断是否是基础类型 
boolean isArray = class1.isArray();//判断是否是集合类
boolean isAnnotation = class1.isAnnotation();//判断是否是注解类 
boolean isInterface = class1.isInterface();//判断是否是接口类 
boolean isEnum = class1.isEnum();//判断是否是枚举类 
boolean isAnonymousClass = class1.isAnonymousClass();//判断是否是匿名内部类 
boolean isAnnotationPresent = class1.isAnnotationPresent(Deprecated.class);//判断是否被某个注解类修饰，这个应该使用率比较高
String className = class1.getName();//获取class名字 包含包名路径 
Package aPackage = class1.getPackage();//获取class的包信息 
String simpleName = class1.getSimpleName();//获取class类名 
int modifiers = class1.getModifiers();//获取class访问权限 
Class<?>[] declaredClasses = class1.getDeclaredClasses();//内部类 
Class<?> declaringClass = class1.getDeclaringClass();//外部类
ClassLoader ClassLoader = class1.getClassLoader() 返回类加载器

getSuperclass()：获取某类所有的父类  
getInterfaces()：获取某类所有实现的接口
```

## 动态代理

动态代理即代理模式在 java 中的实现，通俗说，就是给某个对象提供一个代理对象，并由代理对象控制对于原对象的访问，客户不直接操控原对象，而是通过代理对象间接地操控原对象

代理的操作是通过`java.lang.reflect.Proxy`类中实现的，通过 Proxy 的`newProxyInstance()`方法可以创建一个代理对象

```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler handler)
//loader : 指定代理类对象的加载器
//interfaces : 指定代理类所实现的接口，为了实现一般泛化
//handler : 指定代理对象的handler，当对代理类对象进行方法调用时，会先交给handler进行处理，一般handler对象中拥有原对象
```

操作步骤：

1. 书写代理类和代理方法，在代理方法中实现代理`Proxy.newProxyInstance()`;
2. 代理中需要的参数分别为：被代理的类的类加载器`class.getClassLoader()`，被代理类的所有实现接口`new Class[] { Interface.class }`，句柄方法`new InvocationHandler()`;
3. 在句柄方法中重写`invoke`方法，`invoke`方法的输入有3个参数`Object proxy`（代理类对象）, `Method method`（被代理类的方法）,Object[] args（被代理类方法的传入参数），在这个方法中，我们可以定制化的写我们的业务；
4. 获取代理类，强转成被代理的接口，因为`newProxyInstance`返回的是`Object`对象；
5. 最后，我们可以像没被代理一样，调用接口的任何方法，方法被调用后，方法名和参数列表将被传入代理类的`invoke`方法中，进行新业务的逻辑流程。

具体使用案例：

```java
public interface PersonInterface {
    void doSomething();
    void saySomething();
}

public class PersonImpl implements PersonInterface {
    @Override
    public void doSomething() {
        System.out.println("人类在做事");
    }
    @Override
    public void saySomething() {
        System.out.println("人类在说话");
    }
}

public class PersonProxy {
    public static void main(String[] args) {
        final PersonImpl person = new PersonImpl();
        PersonInterface proxyPerson = (PersonInterface) Proxy.newProxyInstance(PersonImpl.class.getClassLoader(),
                PersonImpl.class.getInterfaces(), new InvocationHandler() {
                    
						  //在下面的invoke方法里面写我们的业务
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        if(method.getName()=="doSomething"){
                            person.doSomething();
                            System.out.println("通过常规方法调用了实现类");
                        }else{
                            method.invoke(person,args);
                            System.out.println("通过反射机制调用了实现类");                        }
                        return null;
                    }
                });
        proxyPerson.doSomething();
        proxyPerson.saySomething();
    }
}
---> 指定interfaces的目标就是，是的返回的代理对象就是该接口类型的对象

```

但是，**以上的动态代理方式有一个致命缺点，newProxyInstance中需要指定接口（原类的接口），也就是如果原类并没有实现某个接口的话就不能使用**，这种JDK动态代理方式，只能对实现了接口的类进行，没有实现接口的类不能使用JDK动态代理。

另一种，`cglib`动态代理，可以解决这个问题！

`cglib`是针对类来实现代理的，他的原理是对指定的目标类生成一个子类，并覆盖其中方法实现增强，但因为采用的是继承，所以不能对final修饰的类进行代理

```java
public class CglibObjectProxy {  
  
    public static Object ceateProxtObject(final Object object,Class clazz) {  
        // 声明增加类实例  
        Enhancer en = new Enhancer();  
        // 设置被代理类字节码，CGLIB根据字节码生成被代理类的子类  
        en.setSuperclass(clazz);  
        // 设置回调函数，即一个方法拦截  
        en.setCallback(new MethodInterceptor() {  
            @Override  
            public Object intercept(Object arg0, Method method, Object[] args,  
                    MethodProxy arg3) throws Throwable {  
                  
                // 注意参数object,仍然为外部声明的源对象，且Method为JDK的Method反射  
                Object o = method.invoke(object, args);  
  
                return o;  
            }  
        });  
        return en.create();  
    }  
      
    public static void main(String[] args) {  
        // 未实现接口的类的代理  
        Person proxyPerson=(Person) CglibObjectProxy.ceateProxtObject(new Person(),Person.class);  
        proxyPerson.active("Talk with sb.");  
        // 实现接口的类的代理  
        IAnimal proxyDog=(IAnimal) CglibObjectProxy.ceateProxtObject(new Dog(),Dog.class);  
        proxyDog.active("Dog lying ...........");  
          
    }  
  
} 
```

注意，`cglib`并不是jdk中自带的库，需要在第三方引入，可以通过 maven:

```xml
<!-- https://mvnrepository.com/artifact/cglib/cglib -->
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>2.2.2</version>
</dependency>
```

对比：

* java动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理。
* cglib动态代理是利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理

SpringAOP就是基于这两种动态代理方式实现，其动态代理策略是：

1. 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP 
2. 如果目标对象实现了接口，可以强制使用CGLIB实现AOP 
3. 如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换

## 注解

注解是什么？ 

* 注解即元数据,就是源代码的元数据
* 注解在代码中添加信息提供了一种形式化的方法,可以在后续中更方便的 使用这些数据
* Annotation是一种应用于类、方法、参数、变量、构造器及包声明中的特殊修饰符。它是一种由JSR-175标准选择用来描述元数据的一种工具。

注解的作用：

* 生成文档
* 跟踪代码依赖性，实现替代配置文件功能,减少配置。如Spring中的一些注解
* 在编译时进行格式检查，如@Override等
* 每当你创建描述符性质的类或者接口时,一旦其中包含重复性的工作，就可以考虑使用注解来简化与自动化该过程，典型如lambok

### Java注解

Java内置的几个注解：

* @Override:表示当前的方法定义将覆盖超类中的方法，如果出现错误，编译器就会报错。
* @Deprecated:如果使用此注解，编译器会出现警告信息。
* @SuppressWarnings:忽略编译器的警告信息。

元注解：负责注解其他注解（自定义注解的时候用到的）

* @Target  说明了Annotation所修饰的对象范围，如类、方法、成员等

|类型				|用途|
|---------------|----|
|CONSTRUCTOR		|用于描述构造器|
|FIELD				|用于描述域|
|LOCAL_VARIABLE	|用于描述局部变量|
|METHOD			|	用于描述方法|
|PACKAGE			|用于描述包|
|PARAMETER		|	用于描述参数|
|TYPE				|用于描述类、接口(包括注解类型) 或enum声明|

如，在自定义注解时：

```
@Target({ElementType.METHOD})
public @interface MyMethodAnnotation {
}

@MyMethodAnnotation 注解只能用于标注方法，如果用于标注类或者成员编译器会报错
```

* @Retention  定义了该Annotation被保留的时间长短

|类型		|用途								|说明|
|---------|-----------------------------|---|
|SOURCE	|在源文件中有效（即源文件保留）		|仅出现在源代码中，而被编译器丢弃|
|CLASS		|在class文件中有效（即class保留）	|被编译在class文件中|
|RUNTIME	|在运行时有效（即运行时保留）		|编译在class文件中|

示例：

```java
@Target({ElementType.TYPE})  //用在描述类、接口或enum
@Retention(RetentionPolicy.RUNTIME)  //运行时有效
public @interface MyClassAnnotation {
    String value();  //这个MyClassAnnotation注解有个value属性，将来可以设置/获取值
}

@MyClassAnnotation 会一直保留到运行期，也就是可以通过反射来获取该标注，可以拿到其中的value等
```


* `@Documented`  描述其它类型的annotation应该被作为被标注的程序成员的公共API，因此可以被例如javadoc此类的工具文档化
* `@Inherited`   阐述了某个被标注的类型是被继承的，即使用了@Inherited修饰的annotation类型被用于一个class,则这个annotation将被用于该class的子类

### 自定义注解

自定义注释的实现形式如下：

```java
public @interface 注解名{
  //...
}
```

自定义注解需要注意的规则：

* 修饰符只能是public 或默认(default)
* 参数成员只能用基本类型byte,short,int,long,float,double,boolean八种基本类型和String,Enum,Class,annotations及这些类型的数组
* **如果只有一个参数成员,最好将名称设为”value”**
* 注解元素必须有确定的值,可以在注解中定义默认值,也可以使用注解时指定,非基本类型的值不可为null,常使用空字符串或0作默认值
* 在表现一个元素存在或缺失的状态时,定义一下特殊值来表示,如空字符串或负值

示例：

```java
@Target(ElementType.FIELD)
@Retention(value=RetentionPolicy.RUNTIME)
@Documented
public @interface MyFieldAnnotation {
    int id() default 0;

    String name() default "";
}
```

参考：

[Java反射、动态代理与注解](https://juejin.im/post/5a44c0ad518825455f2f96e5)

[JDK动态代理和cglib动态代理 - 简书](https://www.jianshu.com/p/1712ef4f2717)
