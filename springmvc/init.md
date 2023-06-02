在web.xml中配置的DispatcherServlet，观察其继承关系：  
DispatcherServlet --> FrameworkServlet --> HttpServletBean --> HttpServlet --> GenericServlet --> interface Servlet  
可以发现它实际上是Tomcat的一个Servlet，而tomcat对Servlet的初始化流程将从init(ServletConfig var1)方法开始：  
---> interface Servlet # void init(ServletConfig var1)  
---> abstract class GenericServlet implements Servlet # void init(ServletConfig config) ---> void init()  
---> abstract class HttpServletBean extends HttpServlet # void init() 创建BeanWrapper获取和设置bean中的属性值，创建resourceLoader读取文件资源  
    ---> void initServletBean()  
---> abstract class FrameworkServlet extends HttpServletBean # void initServletBean()  
    ---> WebApplicationContext initWebApplicationContext() 准备创建WebApplicationContext，提供管理、获取bean的功能  
    ---> WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent)  
    ---> void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) 添加事件监听器SourceFilteringListener  

在springmvc中，提供了ApplicationEventPublisher#publishEvent(Object)（事件发布器）、ApplicationEvent（事件）与 ApplicationListener（事件监听器）。当springmvc通过ApplicationEventPublisher#publishEvent(Object)发布ApplicationEvent（事件）时，ApplicationListener（事件监听器）将会监听到。  

在springmvc初始化过程中，SourceFilteringListener实际上调用的是另一个事件监听器ContextRefreshListener，因此当ApplicationContext容器初始化完成或者被刷新的时候，就会执行ContextRefreshListener的onApplicationEvent方法：  
---> class ContextRefreshListener # void onApplicationEvent(ContextRefreshedEvent event) 其中ContextRefreshListener是FrameworkServlet的内部类  
---> abstract class FrameworkServlet # void onApplicationEvent(ContextRefreshedEvent event)  
    ---> void onRefresh(ApplicationContext context)  
---> class DispatcherServlet extends FrameworkServlet # void onRefresh(ApplicationContext context)  
    ---> void initStrategies(ApplicationContext context)  

```java
DispatcherServlet的initStrategies方法，将会对springmvc的各个组件进行初始化：
    protected void initStrategies(ApplicationContext context) {
        this.initMultipartResolver(context); //文件上传解析器
        this.initLocaleResolver(context); //语言本地化解析器
        this.initThemeResolver(context); //动态样式解析器
        this.initHandlerMappings(context);
        this.initHandlerAdapters(context);
        this.initHandlerExceptionResolvers(context);
        this.initRequestToViewNameTranslator(context);
        this.initViewResolvers(context);
        this.initFlashMapManager(context);
    }
```

----------------------------------------  

```java
private void initHandlerMappings(ApplicationContext context) {
    this.handlerMappings = null;
    //springmvc会加载所有实现了HandlerMapping接口的bean，并进行排序。
    if (this.detectAllHandlerMappings) {
        Map<String, HandlerMapping> matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerMappings = new ArrayList(matchingBeans.values());
            AnnotationAwareOrderComparator.sort(this.handlerMappings);
        }
    } else { //将detectAllHandlerMappings设置为false，springmvc将只加载名为HandlerMapping的bean
        try {
            HandlerMapping hm = (HandlerMapping)context.getBean("handlerMapping", HandlerMapping.class);
            this.handlerMappings = Collections.singletonList(hm);
        } catch (NoSuchBeanDefinitionException var4) {}
    }
    //如果没有自定义HandlerMapping，springmvc会加载默认提供的HandlerMapping
    if (this.handlerMappings == null) {
        this.handlerMappings = this.getDefaultStrategies(context, HandlerMapping.class);
        ...
    }
    ...
}

protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
    if (defaultStrategies == null) {
        try {
            //DispatcherServlet.properties保存着DispatcherServlet的一些默认实现类
            ClassPathResource resource = new ClassPathResource("DispatcherServlet.properties", DispatcherServlet.class);
            defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
        } catch (IOException var15) { ... }
    }
    ...
            try {
                Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
                //创建从DispatcherServlet.properties获取的HandlerMapping默认实现类
                Object strategy = this.createDefaultStrategy(context, clazz);
                strategies.add(strategy);
            } catch (ClassNotFoundException var13) { ... 
            } catch (LinkageError var14) { ... }
    ...
}

protected Object createDefaultStrategy(ApplicationContext context, Class<?> clazz) {
    return context.getAutowireCapableBeanFactory().createBean(clazz);
}
```

DispatcherServlet.properties中提供的HandlerMapping默认实现类有三个：  
org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\  
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\  
	org.springframework.web.servlet.function.support.RouterFunctionMapping  

RequestMappingHandlerMapping的继承关系：  
RequestMappingHandlerMapping --> RequestMappingInfoHandlerMapping --> AbstractHandlerMethodMapping --> InitializingBean  
在前面的context.getAutowireCapableBeanFactory().createBean(clazz)代码中，实际上调用了AbstractAutowireCapableBeanFactory的createBean(Class<T> beanClass)方法，这是spring创建并初始化bean的方法，过程中会调用bean的afterPropertiesSet()方法，而afterPropertiesSet()则实现自InitializingBean接口。  
RequestMappingHandlerMapping并没有重写afterPropertiesSet()方法，因此其创建并初始化过程中，实际上是调用了父类AbstractHandlerMethodMapping的afterPropertiesSet()方法。  

```java
public void afterPropertiesSet() {
    this.initHandlerMethods();
}

protected void initHandlerMethods() {
    String[] var1 = this.getCandidateBeanNames();
    int var2 = var1.length;

    for(int var3 = 0; var3 < var2; ++var3) {
        String beanName = var1[var3];
        if (!beanName.startsWith("scopedTarget.")) {
            this.processCandidateBean(beanName);
        }
    }

    this.handlerMethodsInitialized(this.getHandlerMethods());
}

protected void processCandidateBean(String beanName) {
    Class<?> beanType = null;

    try {
        beanType = this.obtainApplicationContext().getType(beanName);
    } catch (Throwable var4) { ... }

    if (beanType != null && this.isHandler(beanType)) {
        this.detectHandlerMethods(beanName);
    }

}
```
