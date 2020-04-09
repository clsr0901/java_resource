### 1. 环境搭建

* 第一步搭建项目，根据下面项目详情的内容先把project建好，IDEA如何搭建Spring项目，这个靠自己了
* 第二步源码下载，在 [Spring源码下载](https://repo.spring.io/release/org/springframework/spring/ ) 下载对应版本的Spring源码，本文基于4.3.11.RELEASE版本
* 第三部导入源码，如何导入看这篇[Intellij idea关联jar包和源码]( https://blog.csdn.net/wangbin_eamil/article/details/80267124 )

### 2. 项目详情

* Applicaton.java

```java
public class Application {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("classpath:application.xml");
        MessageService messageService = (MessageService) context.getBean("messageService");
        System.out.println(messageService.getMessage());
    }
}
```

* MessageService.java

```java
public interface MessageService {
    String getMessage();
}
```

* MessageServiceImpl.java

```java
public class MessageServiceImpl implements MessageService {
    public String getMessage() {
        return "hello world";
    }
}
```

* application.xml

```java
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd" default-autowire="byName">

    <bean id="messageService" class="com.ktcatv.springmvc.service.impl.MessageServiceImpl"/>
</beans>
```

* pom.xml

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.test</groupId>
    <artifactId>springmvc</artifactId>
    <version>1.0-SNAPSHOT</version>
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>4.3.11.RELEASE</version>
    </dependency>
</dependencies>

</project>
```

以上就是整个项目的所有文件，如果有问题请自行谷歌

