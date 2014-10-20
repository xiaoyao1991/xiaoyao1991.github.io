---
layout: post
title: "Spring-AOP and AspectJ"
date: 2014-10-19 22:50:20 -0500
comments: true
categories: CS Java Spring J2EE AspectJ annotation AOP
---
###Context:  

1. [@annotation & @decorator](#annotationAndDecorator)
2. [Use Case](#usecase)
3. [Spring AOP & AspectJ](#spring-aop)
4. [Code Sample](#code):  
    - [Maven Dependencies](#pom)
    - [Aspects](#aspects)
	- [Annotations](#annotations)
	- [Config](#config)
	- [Service](#service)
	- [Let's run it](#app)
5. [Pitfalls](#pitfalls)

<!-- more -->

<a name="annotationAndDecorator"></a>
###@annotation & @decorator
I was recently looking into Java @annotation and have always been comparing it with Python @decorator. @annotations and @decorators share the same form, but function differently. 

Java @annotation is designed to be meta data marker. It marks certain elements like methods, classes, or variables without changing the actual work of the code. When defining an annotation in Java, a retention policy needs to be specified. The retention policy decides whether if the annotation will exist in the .class files after compilation and should it be accessible during runtime.

There are 3 types of retention policies [1](#r1):  

- CLASS (default): Annotations are to be recorded in the class file by the compiler but need not be retained by the VM at run time.
- SOURCE: Annotations are to be discarded by the compiler.
- RUNTIME: Annotations are to be recorded in the class file by the compiler and retained by the VM at run time, so they may be read reflectively.

One typical example of annotation is @Override. Its retention policy is SOURCE which means it is only there for developers' reference, and will not exist in the compiled files. JUnit @Test annotation, on the other hand, has the RUNTIME retention policy and will exist after compiled. @Test leaves a mark on the test functions and so later JUnit can know which functions are tests and should it run. 

Python @decorator is an implementation of the [Decorator Pattern](http://en.wikipedia.org/wiki/Decorator_pattern). When a decorated method is called, a wrapper function elsewhere that defines the decorator will catch the called method, and so can apply some preprocess/postprocess logics to the decorated method. **In a nutshell, a @decorator is a function that takes a target function as parameter, and then returns a wrapper function that call the target function and also do some extra works around.** Python @decorators are very useful syntactic sugar and are very easy to use. An example decorator looks like this:  

{% codeblock lang:python decorator_example.py %}
# Define a decorator named 'deprecated'
def deprecated(func): 
	def wrapper():
		print 'This method is deprecated!'
		func()
		print 'But you use it anyway huh!'
	return wrapper

@deprecated
def some_deprecated_method():
	print 'I am a deprecated method'
	
some_deprecated_method()
# This method is deprecated!
# I am a deprecated method	
# But you use it anyway huh!
{% endcodeblock %}
---
<a name="usecase"></a>
### Use Case
Decorator Pattern is useful in many cases. For example:  

- Log the running time for certain functions.
- RESTful Web Services usually need some user role filtering(such as user login) on some API services.
- Common constrains on database queries.

Most of the Python Web Framework implemented user login functions by decorators. So the developers only need to decorate the functions that require authentication with the provided decorator. This is very easy and keeps the code clear. I'd like to use Java @annotation to do exactly the same thing, which lead us to Spring-AOP and AspectJ.

---
<a name="spring-aop"></a>
### Spring AOP & AspectJ
Spring AOP(Aspect Oriented Programming) is a technique that enables adding executable blocks to the source code without explicitly changing it. For example, we don't want to add code blocks and conditions inside the API Web Service codes which pollute our code with multiple conditions check. Instead, we want some other classes to intercept these specific API calls, check the authentication and then decide whether to trigger the service behind the scenes. We want that interceptor to understand the annotations and intercept every call to those specific methods with the annotation. This case perfectly fits the original intent of AOP — to avoid re-implementation of some common behavior in multiple classes. In terms of AOP, our solution can be explained as creating an aspect that cross-cuts the code at certain join points and applies an around advice that implements the desired functionality. [2](#r2)

Spring-AOP enables you to use @AspectJ annotation style to make aspects. Note that with proper configuration, Spring-AOP will include the @AspectJ aspects into its management scope, and so we can inject beans into aspects too.

---
<a name="code"></a>
### Show Me The Code
The following code samples use Spring annotation-based configurations instead of XML-based configs. There are many tutorials and resources online regarding implementing such functions in XML-based Spring, but I didn't find much resources on annotation-based Spring. So I'm gonna do it with annotation-based configuration and have a quick walkthrough on the code. More details on Spring annotation-based configuration and Spring-AOP including some terminologies can be found at [3](#r3).

<a name="pom"></a>
#### Maven Dependencies
{% codeblock lang:xml pom.xml %}
<dependencies>      
    <!-- Spring -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>4.1.1.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>4.1.1.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>4.1.1.RELEASE</version>
    </dependency>
    
    <!-- AspectJ -->
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjrt</artifactId>
        <version>1.8.2</version>
    </dependency>
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>1.8.2</version>
    </dependency>
    
    <!-- JavaConfig need this library -->
    <dependency>
        <groupId>cglib</groupId>
        <artifactId>cglib</artifactId>
        <version>2.2.2</version>
    </dependency>
    <dependency>
        <groupId>javax.inject</groupId>
        <artifactId>javax.inject</artifactId>
        <version>1</version>
    </dependency>
</dependencies>
{% endcodeblock %}

<a name="aspects"></a>
#### Aspects
{% codeblock lang:java TestAspect.java %}
package com.xiaoyao.aspects;
import javax.inject.Inject;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;

@Aspect
public class TestAspect {
    @Inject private String appId;
    
    @Around("execution(* *(..)) && @annotation(com.xiaoyao.annotations.TestAnnotation)")
    public Object authenticate(final ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("[TestAspect] Intercepted");
        System.out.println("[TestAspect] appid: " + appId);
        return pjp.proceed();
    }
}
{% endcodeblock %}
This is a simple aspect with a single around advice `around()` inside. The aspect is annotated with @Aspect and advice is annotated with @Around. Annotation @Around has one parameter, which — in this case — says that the advice should be applied to a method if:  

- its visibility modifier is * (public, protected or private);
- its name is name * (any name);
- its arguments are .. (any arguments);
- it is annotated with `@TestAnnotation` which will be defined later;

I also inject a bean `appId` into this aspect. I will discuss the proper configuration required in order to make such injection works.

When a call to an annotated method is to be intercepted, method `around()` executes before executing the actual method. Method `around()` receives an instance of class `ProceedingJoinPoint` and returns an object. `ProceedingJoinPoint` can be seen as the original method, and the return value will be used as the result of the original method. In order to call the original method, the advice has to call `proceed()` of the join point object.


{% codeblock lang:java AdditionalTestAspect.java %}
package com.xiaoyao.aspects;
import javax.inject.Inject;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import com.xiaoyao.annotations.AdditionalTestAnnotation;

@Aspect
public class AdditionalTestAspect {
    @Inject private String appId;
    
    @Around("execution(* *(..)) && @annotation(additionalTestAnnotation)")
    public Object anotherAuthenticate(final ProceedingJoinPoint pjp, final AdditionalTestAnnotation additionalTestAnnotation) throws Throwable {
        System.out.println("[AdditionalTestAspect] Intercepted");
        System.out.println("[AdditionalTestAspect] appid: " + appId);
        System.out.println("[AdditionalTestAspect] annotation: " + additionalTestAnnotation.value());
        return pjp.proceed();
    }
}
{% endcodeblock %}
This is another aspect with a single around advice. The only difference with the `TestAspect` is that this aspect will read the attribute from the annotation. 

<a name="annotation"></a>
#### Annotations

{% codeblock lang:java TestAnnotation.java %}
package com.xiaoyao.annotations;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.METHOD, ElementType.TYPE })
public @interface TestAnnotation {}
{% endcodeblock %}

{% codeblock lang:java AdditionalTestAnnotation.java %}
package com.xiaoyao.annotations;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.METHOD, ElementType.TYPE })
public @interface AdditionalTestAnnotation {
    int value();
}
{% endcodeblock %}

<a name="config"></a>
#### Config
{% codeblock lang:java AppConfig.java %}
package com.xiaoyao.config;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import com.xiaoyao.aspects.AdditionalTestAspect;
import com.xiaoyao.aspects.TestAspect;

@ComponentScan(basePackages = {"com.xiaoyao"})
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {

	@Bean
    public String appId() {
	    return "Xiaoyao's App";
    }
	
    @Bean
	public TestAspect testAspect() {
    	return new TestAspect();
    }

	@Bean
    public AdditionalTestAspect additionalTestAspect() {
	    return new AdditionalTestAspect();
    }
}
{% endcodeblock %}
This is a standard configuration POJO for annotation-based Spring. In order to use annotation-based configuration, we just need to annotate a config POJO with `@Configuration` and `ComponentScan`, and annotate each beans with `@Bean`. 

`EnableAspectJAutoProxy` is required for Spring to auto discover @AspectJ aspects, and perform runtime weaving [3](#r3). **In addition, the aspects also have to be declared as beans so that Spring will include them into its management scope**. With this configuration, our aspects and annotations are ready to go. 

<a name="service"></a>
#### Service
{% codeblock lang:java TestService.java %}
package com.xiaoyao.services;
import org.springframework.stereotype.Service;
import com.xiaoyao.annotations.AdditionalTestAnnotation;
import com.xiaoyao.annotations.TestAnnotation;

@Service
public class TestService {
    @TestAnnotation()
    @AdditionalTestAnnotation(1)
    public void serve() {
        System.out.println("This is a service");
    }
}
{% endcodeblock %}
In the service class, we simply annotate the method with our custom annotations and this method will be intercepted when it gets called.

<a name="app"></a>
#### Let's run it...
{% codeblock lang:java App.java %}
package com.xiaoyao.app;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import com.xiaoyao.services.TestService;
import com.xiaoyao.config.AppConfig;

public class App {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        TestService testService = (TestService) context.getBean("testService");
        testService.serve();
        /*
            Output:
            [TestAspect] Intercepted
            [TestAspect] appId: Xiaoyao's App
            [AdditionalTestAspect] Intercepted
            [AdditionalTestAspect] appId: Xiaoyao's App
            [AdditionalTestAspect] annotation: 1
            This is a service
        */
    }
}
{% endcodeblock %}

---
<a name="pitfalls"></a>
### Pitfalls

- @annotation can have attributes, but attribute type can only be primitives and Class<>.
- @annotation attributes can only be assigned with constant expression. Namely, you have to do `@AdditionalTestAnnotation(1)` instead of `@AdditionalTestAnnotation(this.id)` even if `this.id` is a constant.


---
### Reference:  

1. <a name="r1"></a>[JavaDoc](http://docs.oracle.com/javase/7/docs/api/java/lang/annotation/RetentionPolicy.html)
2. <a name="r2"></a>[Java Method Logging with AOP and Annotations](http://www.yegor256.com/2014/06/01/aop-aspectj-java-method-logging.html)
3. <a name="r3"></a>[Spring Doc](http://docs.spring.io/spring-framework/docs/current/spring-framework-reference/html/aop.html)