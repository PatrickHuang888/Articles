# ADP上混合Scala,Java和Spring开发spark应用的方法

spark的弹性分布式数据集RDD(Resilient Distributed Dataset)提供了丰富的数据处理函数，
例如map,reduce,filter等，Scala利用函数式编程的写法使用这些函数是非常方便的，代码简洁高效，用Java则繁琐很多。
但是我们开发企业应用中不可能有这么单纯的环境，需要和各种系统集成，Scala就非常吃力，
而Java已经发展出了很多成熟高效的环境，例如Spring
想象这样一个场景，在操作一系列数据后，想要把结果写到MongoDB中去，
如果用Scala就要自己手动写读写数据库语句，非常麻烦。
而Spring就可以做到完全的对象操作。

那么我们怎么能够做到把Scala,Java,Spring集成在一个项目里开发呢？
步骤如下：
### 1 项目管理配置
我们使用Maven管理项目，配置如下：
```xml
    <!--引入Spring Boot -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.1.RELEASE</version>
    </parent>

        <dependencies>
            <!--引入Spring data Mongodb的支持-->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-data-mongodb</artifactId>
            </dependency>

            <!--引入spark-->
            <dependency>
                <groupId>org.apache.spark</groupId>
                <artifactId>spark-core_${scala.binary.version}</artifactId>
                <version>1.6.2</version>
            </dependency>
            <dependency>
                <groupId>org.apache.spark</groupId>
                <artifactId>spark-sql_${scala.binary.version}</artifactId>
                <version>1.6.2</version>
            </dependency>
        </dependencies>

    <build>
        <plugins>
            <!--引入Scala编译-->
            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <version>3.2.2</version>
                <executions>
                    <execution>
                        <id>scala-compile-first</id>
                        <phase>process-resources</phase>
                        <goals>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <scalaCompatVersion>${scala.binary.version}</scalaCompatVersion>
                    <scalaVersion>${scala.version}</scalaVersion>
                    <recompileMode>incremental</recompileMode>
                    <useZincServer>true</useZincServer>
                    <args>
                        <arg>-unchecked</arg>
                        <arg>-deprecation</arg>
                        <arg>-feature</arg>
                    </args>
                </configuration>
            </plugin>

            <!--引入Spring打包-->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```

### 2Java操作数据库
首先
在目录src/main/java目录下按你的对象模型写数据对象代码,如
src/main/java/com/apusic/model/Person.java
然后创建Repository接口,
```java
public interface PersonRepository extends MongoRepository<Person, String>{
}
```
在resource目录下配置application.properties等，这些和一般Java项目并无不同，不做详细描述

### 3Scala编写spark任务并操作数据库存取
在目录src/main/scala目录下创建Scala代码
```scala
package com.apusic.demo

@SpringBootApplication
class TestApplication @Autowired()(personRepository: PersonRepository) extends CommandLineRunner {

  override def run(args: String*): Unit = {
    val sparkContext = new SparkContext()
    val sqlContext = new SQLContext(sparkContext)
    val df= sqlContext.read.parquet(data)
    val rdd= df.map {
        row =>
            ...
    }
    //做一系列你需要的操作
    ...
    //最后存取数据
    jobsRepository.save(person)
  }
}
```
这里的技巧就是怎样使用Spring的注释(Annotation)`Autowried`将repository注入进来

### 4 组装环境
```scala
object TestApplication extend App {
    SpringApplication.run(classOf[TestMongoAccess], args: _*)
}
```
这里关键的把Scala程序和Spring环境挂接在了一起

### 5 运行
编译打包
`mvn package`
得益于Spring boot提供的整套运行环境，我们只需要一个很简单的命令就可以运行这样一个复杂的程序了
`java -jar target/testapplication.jar`