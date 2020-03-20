---
layout:     post
title:      Spring Boot多级配置注入
subtitle:   Spring Boot多级配置注入
date:       2020-03-20
author:     dengcaoyu
header-img: https://i.loli.net/2020/03/20/31dqoJGBkFOg9DA.jpg
catalog: true
tags:
    - Spring Boot
---


在 **Spring Boot** 项目中，我们通常将大量的参数配置在 application.properties 或 application.yml 文件中，通过 `@ConfigurationProperties` 注解，我们可以方便的将配置一次性注入到Bean中，不需要再通过`@Value`注解一个一个的配置。但是当我们的配置是多级时，代码又该如何写呢？

我们先来看一般情况下，我们会如何写
> application.yml

```
person:
  lastname: dengcaoyu
  age: 21
  phone:
    brand: iphone
    color: black
```

> PersonDTO.java

```
@Data
@Configuration
// 在这里“person”必须为配置属性前的全前缀
@ConfigurationProperties(prefix = "person")
public class PersonDTO {

    private String lastname;
    private int age;
    private phone phone;

    @Data
    public class phone {
        private String brand;
        private String color;
    }

}
```
> 测试类

在这里我封装了一层

```
public class SimpleTest extends SpringbootDemoApplicationTests{

    @Autowired
    private PersonDTO personDTO;

    @Test
    public void test() {
        System.out.println(personDTO);
    }

}
```
**输出结果**
![image](https://i.loli.net/2020/03/20/7KbZURz6CfgBEHr.png)
在这里PersonDTO类的phone属性为空，说明**多级配置注入并不能通过配置名自动注入**。那我们应该怎么做呢？

我们对**PersonDTO**类稍作修改

```
@Data
@Configuration
@ConfigurationProperties(prefix = "person")
public class PersonDTO {

    private String lastname;
    private int age;
    
    @Autowired
    private phone phone;

    @Data
    @Configuration
    @ConfigurationProperties(prefix = "person.phone")
    public class phone {
        private String brand;
        private String color;
    }

}

```

![image](https://i.loli.net/2020/03/20/PML4tgpiqRDE8NA.png)
可以看到，注入成功。其实我们只是以同样的方式定义了一个phone配置类，再通过`@Autowired`属性注入就行了。

创造不易，如果对你有帮助的话，就点 [Star](https://github.com/dengcaoy/dengcaoy.github.io) 支持我一下吧！: )