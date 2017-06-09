---
layout: post
title: spring annotation
tags:
- spring annotation
categories: spring
description: spring annotation收集。
---
### 1.  @ComponentScan
Configures component scanning directives for use with @Configuration classes. Provides support parallel with Spring XML's <context:component-scan> element. 

Either basePackageClasses or basePackages (or its alias value) may be specified to define specific packages to scan.*** If specific packages are not defined, scanning will occur from the package of the class that declares this annotation. ***

Note that the <context:component-scan> element has an annotation-config attribute; however, this annotation does not. This is because in almost all cases when using @ComponentScan, default annotation config processing (e.g. processing @Autowired and friends) is assumed. Furthermore, when using AnnotationConfigApplicationContext, annotation config processors are always registered, meaning that any attempt to disable them at the @ComponentScan level would be ignored.

### 2. @Configuration
Indicates that a class declares one or more @Bean methods and may be processed by the Spring container to generate bean definitions and service requests for those beans at runtime, for example: 

 ```java
@Configuration
 public class AppConfig {

     @Bean
     public MyBean myBean() {
         // instantiate, configure and return bean ...
     }
 }
```
Bootstrapping @Configuration classes
Via AnnotationConfigApplicationContext
@Configuration classes are typically bootstrapped using either AnnotationConfigApplicationContext or its web-capable variant, AnnotationConfigWebApplicationContext. A simple example with the former follows: 
```java
 AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
 ctx.register(AppConfig.class);
 ctx.refresh();
 MyBean myBean = ctx.getBean(MyBean.class);
 // use myBean ...
```
 
See AnnotationConfigApplicationContext Javadoc for further details and see AnnotationConfigWebApplicationContext for web.xml configuration instructions. 
Via Spring <beans> XML
As an alternative to registering @Configuration classes directly against an AnnotationConfigApplicationContext, @Configuration classes may be declared as normal <bean> definitions within Spring XML files: 

```xml
 <beans>
    <context:annotation-config/>
    <bean class="com.acme.AppConfig"/>
 </beans>
```
In the example above, <context:annotation-config/> is required in order to enable ConfigurationClassPostProcessor and other annotation-related post processors that facilitate handling @Configuration classes. 
Via component scanning
@Configuration is meta-annotated with @Component, therefore @Configuration classes are candidates for component scanning (typically using Spring XML's <context:component-scan/> element) and therefore may also take advantage of @Autowired/@Inject like any regular @Component. In particular, if a single constructor is present autowiring semantics will be applied transparently: 

```java
 @Configuration
 public class AppConfig {
     private final SomeBean someBean;

     public AppConfig(SomeBean someBean) {
         this.someBean = someBean;
     }

     // @Bean definition using "SomeBean"

 }
```
@Configuration classes may not only be bootstrapped using component scanning, but may also themselves configure component scanning using the @ComponentScan annotation: 

```java
 @Configuration
 @ComponentScan("com.acme.app.services")
 public class AppConfig {
     // various @Bean definitions ...
 }
```
See the @ComponentScan javadoc for details. 
Working with externalized values
Using the Environment API
Externalized values may be looked up by injecting the Spring org.springframework.core.env.Environment into a @Configuration class the usual (e.g. using the @Autowired annotation): 
 ```java
@Configuration
 public class AppConfig {

     @Autowired Environment env;

     @Bean
     public MyBean myBean() {
         MyBean myBean = new MyBean();
         myBean.setName(env.getProperty("bean.name"));
         return myBean;
     }
 }
```
Properties resolved through the Environment reside in one or more "property source" objects, and @Configuration classes may contribute property sources to the Environment object using the @PropertySources annotation: 
 ```java
@Configuration
 @PropertySource("classpath:/com/acme/app.properties")
 public class AppConfig {

     @Inject Environment env;

     @Bean
     public MyBean myBean() {
         return new MyBean(env.getProperty("bean.name"));
     }
 }
```
See Environment and @PropertySource Javadoc for further details. 
Using the @Value annotation
Externalized values may be 'wired into' @Configuration classes using the @Value annotation: 
 ```java
@Configuration
 @PropertySource("classpath:/com/acme/app.properties")
 public class AppConfig {

     @Value("${bean.name}") String beanName;

     @Bean
     public MyBean myBean() {
         return new MyBean(beanName);
     }
 }
```
This approach is most useful when using Spring's PropertySourcesPlaceholderConfigurer, usually enabled via XML with <context:property-placeholder/>. See the section below on composing @Configuration classes with Spring XML using @ImportResource, see @Value Javadoc, and see @Bean Javadoc for details on working with BeanFactoryPostProcessor types such as PropertySourcesPlaceholderConfigurer. 
Composing @Configuration classes
With the @Import annotation
@Configuration classes may be composed using the @Import annotation, not unlike the way that <import> works in Spring XML. Because @Configuration objects are managed as Spring beans within the container, imported configurations may be injected the usual way (e.g. via constructor injection): 

```java
 @Configuration
 public class DatabaseConfig {

     @Bean
     public DataSource dataSource() {
         // instantiate, configure and return DataSource
     }
 }

 @Configuration
 @Import(DatabaseConfig.class)
 public class AppConfig {

     private final DatabaseConfig dataConfig;

     public AppConfig(DatabaseConfig dataConfig) {
         this.dataConfig = dataConfig;
     }

     @Bean
     public MyBean myBean() {
         // reference the dataSource() bean method
         return new MyBean(dataConfig.dataSource());
     }
 }
```
Now both AppConfig and the imported DatabaseConfig can be bootstrapped by registering only AppConfig against the Spring context: 
 new AnnotationConfigApplicationContext(AppConfig.class);
With the @Profile annotation
@Configuration classes may be marked with the @Profile annotation to indicate they should be processed only if a given profile or profiles are active: 
```java
 @Profile("embedded")
 @Configuration
 public class EmbeddedDatabaseConfig {

     @Bean
     public DataSource dataSource() {
         // instantiate, configure and return embedded DataSource
     }
 }

 @Profile("production")
 @Configuration
 public class ProductionDatabaseConfig {

     @Bean
     public DataSource dataSource() {
         // instantiate, configure and return production DataSource
     }
 }
```
See the @Profile and org.springframework.core.env.Environment javadocs for further details. 
With Spring XML using the @ImportResource annotation
As mentioned above, @Configuration classes may be declared as regular Spring <bean> definitions within Spring XML files. It is also possible to import Spring XML configuration files into @Configuration classes using the @ImportResource annotation. Bean definitions imported from XML can be injected the usual way (e.g. using the Inject annotation): 
 ```java
@Configuration
 @ImportResource("classpath:/com/acme/database-config.xml")
 public class AppConfig {

     @Inject DataSource dataSource; // from XML

     @Bean
     public MyBean myBean() {
         // inject the XML-defined dataSource bean
         return new MyBean(this.dataSource);
     }
 }
```
With nested @Configuration classes
@Configuration classes may be nested within one another as follows: 
 ```java
@Configuration
 public class AppConfig {

     @Inject DataSource dataSource;

     @Bean
     public MyBean myBean() {
         return new MyBean(dataSource);
     }

     @Configuration
     static class DatabaseConfig {
         @Bean
         DataSource dataSource() {
             return new EmbeddedDatabaseBuilder().build();
         }
     }
 }
```
When bootstrapping such an arrangement, only AppConfig need be registered against the application context. By virtue of being a nested @Configuration class, DatabaseConfig will be registered automatically. This avoids the need to use an @Import annotation when the relationship between AppConfig DatabaseConfig is already implicitly clear. 
Note also that nested @Configuration classes can be used to good effect with the @Profile annotation to provide two options of the same bean to the enclosing @Configuration class. 

Configuring lazy initialization
By default, @Bean methods will be eagerly instantiated at container bootstrap time. To avoid this, @Configuration may be used in conjunction with the @Lazy annotation to indicate that all @Bean methods declared within the class are by default lazily initialized. Note that @Lazy may be used on individual @Bean methods as well. 

Testing support for @Configuration classes
The Spring TestContext framework available in the spring-test module provides the @ContextConfiguration annotation, which as of Spring 3.1 can accept an array of @Configuration Class objects: 
 ```java
@RunWith(SpringJUnit4ClassRunner.class)
 @ContextConfiguration(classes={AppConfig.class, DatabaseConfig.class})
 public class MyTests {

     @Autowired MyBean myBean;

     @Autowired DataSource dataSource;

     @Test
     public void test() {
         // assertions against myBean ...
     }
 }
```
See TestContext framework reference documentation for details. 
Enabling built-in Spring features using @Enable annotations
Spring features such as asynchronous method execution, scheduled task execution, annotation driven transaction management, and even Spring MVC can be enabled and configured from @Configuration classes using their respective "@Enable" annotations. See @EnableAsync, @EnableScheduling, @EnableTransactionManagement, @EnableAspectJAutoProxy, and @EnableWebMvc for details.

### 3. @Service
Indicates that an annotated class is a "Service", originally defined by Domain-Driven Design (Evans, 2003) as "an operation offered as an interface that stands alone in the model, with no encapsulated state." 

May also indicate that a class is a "Business Service Facade" (in the Core J2EE patterns sense), or something similar. This annotation is a general-purpose stereotype and individual teams may narrow their semantics and use as appropriate. 

This annotation serves as a specialization of @Component, allowing for implementation classes to be autodetected through classpath scanning.

### 4.  @Repository
Indicates that an annotated class is a "Repository", originally defined by Domain-Driven Design (Evans, 2003) as "a mechanism for encapsulating storage, retrieval, and search behavior which emulates a collection of objects".

Teams implementing traditional J2EE patterns such as "Data Access Object" may also apply this stereotype to DAO classes, though care should be taken to understand the distinction between Data Access Object and DDD-style repositories before doing so. This annotation is a general-purpose stereotype and individual teams may narrow their semantics and use as appropriate.

A class thus annotated is eligible for Spring DataAccessException translation when used in conjunction with a PersistenceExceptionTranslationPostProcessor. The annotated class is also clarified as to its role in the overall application architecture for the purpose of tooling, aspects, etc. 

As of Spring 2.5, this annotation also serves as a specialization of @Component, allowing for implementation classes to be autodetected through classpath scanning.




