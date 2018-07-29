# Maven 实践出真知
Apache Maven，是一个软件（特别是Java软件）项目管理及自动构建工具

什么是构建工具：

* 生成源码（如果项目使用自动生成源码）；
* 从源码生成项目文档；
* 编译源码；
* 将编译后的代码打包成JAR文件或者ZIP文件；
* 将打包好的代码安装到服务器、仓库或者其它的地方；

## Maven 仓库
* 中央仓库
	
	Maven的中央仓库由Maven社区提供。默认情况下所有不在本地仓库中的依赖都回去这个中央仓库查找，然后下载到本地仓库。
* 远程仓库
	
	位于服务器的仓库，Maven可以从该仓库下载依赖
* 本地仓库
	
	个人电脑的仓库，存放下载的依赖

可以在pom文件里配置远程仓库。将以下的xml片段放到属性之后：

```xml
<repositories>
   <repository>
       <id>example</id>
       <url>http://example.com/maven2/lib</url>
   </repository>
</repositories>
```

## Maven create project

创建项目示例命令：

`mvn archetype:generate -DgroupId=org.yourcompany.project -DartifactId=application`

生成的项目结构如下：

![](../images/maven_proj_structure.jpg)

标准目录结构：

```
- src
  - main
    - java
    - resources
    - webapp
  - test
    - java
    - resources

- target
```


## Maven architecture

![](../images/maven_architecture.jpg)

以下为一些基本的maven执行命令，build期间执行的就是这些命令


| maven操作 | 功能用途 |
|----------|---------|
| clean | delete target directory |
| validate |validate, if the project is correct|
| compile |compile source code, classes stored in target/classes|
| test |run tests|
| package |take the compiled code and package it in its distributable format, e.g. JAR, WAR|
| verify |run any checks to verify the package is valid and meets quality criteria|
| install |install the package into the local repository|
| deploy |copies the final package to the remote repository|




## Maven Plugin

Maven 默认情况下，会根据几个常用操作选择默认的插件：

|plugin				|			function	| life cycle phase |
|-------------------|-------------------|------------------|
|maven-clean-plugin	|		清理上一次执行创建的目标文件 |clean |
|maven-resources-plugin	|	处理源资源文件和测试资源文件 |resources,testResources|
|maven-compiler-plugin	|	编译源文件和测试源文件      |compile,testCompile|
|maven-surefire-plugin	|		执行测试文件           |test|
|maven-jar-plugin		|		创建 jar	           |jar |
|maven-install-plugin	|		安装 jar，将生成的jar安装到.m2/repository下面|install|
|maven-deploy-plugin	|		发布 jar	|deploy|

## Maven Profile
针对不同的环境采用不同的配置，如开发环境和生产环境使用不一样的配置参数

```xml
<profiles>
    <profile>
      <id>test</id>
      <activation>...</activation> 
      <build>...</build>
      <modules>...</modules>
      <repositories>...</repositories>
      <pluginRepositories>...</pluginRepositories>
      <dependencies>...</dependencies>
      <reporting>...</reporting>
      <dependencyManagement>...</dependencyManagement>
      <distributionManagement>...</distributionManagement>
    </profile>
  </profiles>
```

多profile选择示例：

```xml
<profiles>
    <profile>
        <!-- 本地开发环境 -->
        <id>dev</id>
        <properties>
            <profiles.active>dev</profiles.active>
        </properties>
        <activation>
            <!-- 设置默认激活这个配置 -->
            <activeByDefault>true</activeByDefault>
        </activation>
 		   <build>  
              <filters>  
                  <filter>src/main/resources/dev.properties</filter>  
              </filters>  
          </build>  
    </profile>
    <profile>
        <!-- 发布环境 -->
        <id>release</id>
        <properties>
            <profiles.active>release</profiles.active>
        </properties>
		  <build>  
             <filters>  
                 <filter>src/main/resources/release.properties</filter>  
             </filters>  
         </build>  
    </profile>
    <profile>
        <!-- 测试环境 -->
        <id>beta</id>
        <properties>
            <profiles.active>beta</profiles.active>
        </properties>
 		  <build>  
              <filters>  
                  <filter>src/main/resources/beta.properties</filter>  
              </filters>  
         </build>  
    </profile>
</profiles> 

	<build>  
	<!--该元素设置了项目源码目录，当构建项目的时候，构建系统会编译目录里的源码。该路径是相对于pom.xml的相对路径。-->
        <finalName>myweb</finalName>  
        <resources>  
            <resource>  
                <directory>src/main/resources</directory>  
                <includes>  
                    <include>**/*</include>  
                </includes>  
				   <!--是否使用参数值代替参数名。参数值取自properties元素或者文件里配置的属性，文件在filters元素里列出。--> 
                <filtering>true</filtering>  
                <excludes>  
                    <exclude>application-text.xml</exclude>  
                    <!--<exclude>src/main/resources/application-text.xml</exclude>-->  
                </excludes>  
            </resource>  
        </resources>  
        <!-- <filters> 可以直接在profile的build中配置filter，也可以采用如下方式-->  
            <!--<filter>src/main/resources/${profiles.active}.properties</filter>-->  
        <!--</filters>-->  
    </build>  
```

通过` <activeByDefault>true</activeByDefault>`指定默认环境为dev

1. **profiles**：定义各个环境的变量配置，我上面的代码中有三个环境，所以配了3个profile
	* `<id>`: profile的标示
	* `<properties>`: 自己定义的一些属性，可有可无，比如我配置的jdbc.url这些属性，如果不想通过properties定义这些，可以在改属性下面配置
	* `<filters>`: 比较重要，指定当前profile环境下，属性文件路径；

2. **build** 的配置

* defaultGoal，执行构建时默认的goal或phase，如`jar:jar`或者package等
* directory，构建的结果所在的路径，默认为`${basedir}/target`目录
* finalName，构建的最终结果的名字，该名字可能在其他 plugin 中被改变
* `<resource>`下面的属性
	* `<directory>`: 资源文件的路径，默认位于`${basedir}/src/main/resources/`目录下，配置目录下的文件通过`${key}`会被替换成属性值
	* `<includes>`：一组文件名的匹配模式，被匹配的资源文件将被构建过程处理
	* `<filtering>`：构建过程中是否对资源进行过滤，默认false。即用环境变量、pom文件里定义的属性和指定配置文件里的属性替换属性(*.properties)文件里的占位符(如：`${jdbc.url}`)。
	* `<exclueds>`：一组文件名的匹配模式，被匹配的资源文件将被构建过程忽略。同时被includes和excludes匹配的资源文件，将被忽略。
* `<filters>`：给出对资源文件进行过滤的属性文件的路径（即用于参数值属性替换的数据来源）。这里的filters与`<profile>`的 filter 意思一样，都是指属性文件地址，这个如果上面定义`<profile>`的时候指定了，这里就不需要了，但有些开发习惯是在`<profile>`不定义，然后在`<build>`里指定。

**那么如何选择不同的配置环境呢？**

`mvn clean package -PprofileId`   通过指定`profileId`即可

`activation`：声明一些需要满足的条件，即当满足这些条件的时候，当前profile才被激活
如：

```xml
<profiles>
    <profile>
      <id>test</id>
      <activation>
        <activeByDefault>false</activeByDefault>
        <jdk>1.5</jdk>
        <os>
          <name>Windows XP</name>
          <family>Windows</family>
          <arch>x86</arch>
          <version>5.1.2600</version>
        </os>
        <property>
          <name>sparrow-type</name>
          <value>African</value>
        </property>
        <file>
          <exists>${basedir}/file2.properties</exists>
          <missing>${basedir}/file1.properties</missing>
        </file>
      </activation>
      ...
    </profile>
  </profiles>
```

## Maven build & test

参照：

[Maven Cheat Sheet zeroturnaround.com](https://zeroturnaround.com/rebellabs/maven-cheat-sheet/)

[Maven pom.xml 配置详解 - CSDN博客](https://blog.csdn.net/ithomer/article/details/9332071)

[Maven – POM Reference](http://maven.apache.org/pom.html)
