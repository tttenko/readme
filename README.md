```java
fun main(args: Array<String>) {
    val context = runApplication<Application>(*args) {
        applicationStartup = FlightRecorderApplicationStartup()
    }

    val beanFactory = context.beanFactory
    val lazyBeanCount = beanFactory.beanDefinitionNames.count { beanFactory.getBeanDefinition(it).isLazyInit }

    println("Lazy initialization: ${context.environment.getProperty("spring.main.lazy-initialization")}")
    println("Lazy bean definitions: $lazyBeanCount / ${beanFactory.beanDefinitionCount}")
    println("Initialized singletons: ${beanFactory.singletonCount}")
}
```
