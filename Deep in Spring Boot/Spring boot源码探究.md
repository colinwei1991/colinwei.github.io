Spring boot应用启动过程分析

@SpringBootApplication注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    ...
}
```

整合了三个注解@SpringBootConfiguration、@EnableAutoConfiguration和@ComponentScan

@SpringBootConfiguration源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {
    ...
}
```

就是声明了这是一个Configuration，可以被容器发现并自动加载配置

@EnableAutoConfiguration源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ...
}
```

@EnableAutoConfiguration完成了两个功能，一个是@AutoConfigurationPackage注解实现的，把该类所在包注册到容器中。另一个是引入了AutoConfigurationImportSelector的配置。

先看第一个，@AutoConfigurationPackage是怎么完成包注册的？

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
}
```

