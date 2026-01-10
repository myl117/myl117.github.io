+++
title = "Spring Beans for Absolute Beginners"
date = "2025-09-19"
tags = [
    "java",
    "springboot",
    "spring",
    "beans"
]
+++

If you’ve ever dabbled in the Java Spring framework specifically Spring Boot 🍃, chances are you’ve come across the term beans. Sure, you could pretend they don’t exist and temporarily dodge the headache of figuring them out but let’s be honest, that only delays the inevitable. Don’t worry, though: this article is your guide out of that loop. By the end, you’ll have a clear understanding of what Spring beans 🫘 are and how their lifecycle works. No more guesswork, no more suffering.

One of the core principles of the Spring framework is Inversion of Control (IoC). Simply put, IoC flips the script: instead of you managing the creation and lifecycle of objects, Spring takes the wheel.

In plain Java, you might write something like this:

```java
Car car = new Car();

car.init();
car.drive();
car.cleanup();
```

Here, you’re in full control. You create the object, initialize it and clean it up. With Spring Boot’s IoC, you declare your dependencies, and the Spring IoC container (aka ApplicationContext) handles object creation, lifecycle and injection for you automatically thanks to Dependency Injection (DI).

## Spill the Beans

![Spring Beans meme](/images/spring-beans-absolute-beginners/meme.jpg)

Now that we’ve got a handle on the Spring IoC container, let’s talk beans. Simply put, a bean is an object that the Spring IoC container creates and manages for you. Instead of manually writing new Car() or new Engine(), Spring takes care of creating those objects and storing them in the container. When you need one, you can grab it using @Autowired or @Qualifier.

Let’s put this into practice with a demo Spring Boot controller that uses a bean. To follow along, you’ll need a new Spring Boot project or you can use an existing one. First, we’ll create our bean: make a new folder called components, then create a Car.java file. Inside, define your Car class. To let Spring know this isn’t just a regular POJO (Plain Old Java Object), annotate it with @Component. Here’s a full example including lifecycle annotations, which Spring calls automatically through the IoC container:

```java
@Component
public class Car {
  @PostConstruct
  public void init() {
    System.out.println("Car initialized");
  }

  public void drive() {
    System.out.println("Car driving");
  }

  @PreDestroy
  public void cleanup() {
    System.out.println("Car cleaned up");
  }
}
```

Next, we’ll create a controller with a route to call our drive method. You can test it in the browser to make sure everything is set up correctly.

```java
@RestController
@RequestMapping("/api")
public class CarController {
  private final Car car;

  public CarController(Car car) {
    this.car = car;
  }

  @GetMapping("/car")
  public String getMethodName() {
    car.drive();
    return "Vrooom 🏎️💨";
  }
}
```

In our controller, we define a constructor that takes a Car object as a parameter. So what’s happening under the hood? When Spring creates the CarController bean, it sees that the constructor requires a Car. It then digs into its IoC container and automatically injects the Car bean for us. This is an example of dependency injection, specifically constructor injection where Spring manages dependencies so you don’t have to.

## @Component vs @Bean

So, what’s the difference between these two annotations? 🤌

When we use @Component, we place the annotation directly on the class we want Spring to manage. For example:

```java
@Component
public class EngineService {
    public void start() {
        System.out.println("*engine noises*");
    }
}
```

Here, the bean is the class itself. Spring performs a component scan, finds the class, and turns it into a managed bean automatically.

On the other hand, @Bean works differently. Instead of annotating the class itself, we create a factory method inside an @Configuration class that returns the object. This approach is useful when we can’t modify the original class like a third-party library for a payment processor.

```java
@Configuration
public class PaymentConfig {
    @Bean
    public PayPalClient payPalClient() {
        PayPalClient client = new PayPalClient();
        client.setApiKey("MY_SECRET_KEY");
        return client;
    }
}
```

## Bean Scope

Bean scope defines how long a bean lives and how many instances Spring creates in its IoC container. By default, all beans are singletons, meaning there’s only one instance in the container, shared wherever it’s injected. If you need a different behavior, you can use the @Scope annotation. Here are some of the main scopes you’ll encounter in Spring:

| Scope         | Lifetime            | Instances           |
| ------------- | ------------------- | ------------------- |
| **singleton** | Entire application  | 1 per IoC container |
| **prototype** | Until GC removes it | New every time      |
| **request**   | One HTTP request    | 1 per request       |
| **session**   | User session        | 1 per session       |

Let’s switch our Car bean from singleton to request scope:

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class Car {
  @PostConstruct
  public void init() {
    System.out.println("Car initialised");
  }

  public void drive() {
    System.out.println("Car driving");
  }

  @PreDestroy
  public void cleanup() {
    System.out.println("Car cleaned up");
  }
}
```

This tells Spring to create a new instance of the Car bean for each request. But what about proxyMode = ScopedProxyMode.TARGET_CLASS? Well, Spring can’t inject a request-scoped bean directly into a singleton like CarController, because the singleton is created at application startup. Instead, Spring injects a proxy object. When a method is called on the proxy, Spring provides the actual request-scoped bean needed for that specific request.

To make sure our new scope is working, we can take a look at the console and see that we are getting new logs every time we make a request.

```ini
Car initialized
Car driving
Car cleaned up

Car initialized
Car driving
Car cleaned up

...
```

And that’s all the essential theory a beginner needs to know about Spring Beans. Of course, this article only scratches the surface and there’s a lot more to explore when it comes to dependency injection and the inner workings of Spring Beans. I may dive deeper in future articles, but for now, I hope this gives you a solid launchpad 🚀 to start using beans confidently in your Spring applications!
