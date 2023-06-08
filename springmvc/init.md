在web.xml中配置的DispatcherServlet，观察其继承关系：  
DispatcherServlet --> FrameworkServlet --> HttpServletBean --> HttpServlet --> GenericServlet --> interface Servlet  
可以发现它实际上是Tomcat的一个Servlet，而tomcat对Servlet的初始化流程将从init(ServletConfig var1)方法开始：  
```java
public interface Servlet {
    void init(ServletConfig var1) throws ServletException;
    ...
}

public abstract class GenericServlet implements Servlet, ServletConfig, Serializable {
    public void init(ServletConfig config) throws ServletException {
        this.config = config;
        this.init();
    }
    public void init() throws ServletException {}
    ...
}

public abstract class HttpServletBean extends HttpServlet implements EnvironmentCapable, EnvironmentAware {
    public final void init() throws ServletException {
        //创建BeanWrapper获取和设置bean中的属性值，创建resourceLoader读取文件资源
        ...
        this.initServletBean();
    }
    protected void initServletBean() throws ServletException {}
    ...
}

public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
    protected final void initServletBean() throws ServletException {
        ...
        try {
            //准备创建WebApplicationContext，提供管理、获取bean的功能
            this.webApplicationContext = this.initWebApplicationContext();
            this.initFrameworkServlet();
        } catch (RuntimeException | ServletException var4) { ... }
        ...
    }
    protected WebApplicationContext initWebApplicationContext() {
        WebApplicationContext rootContext = WebApplicationContextUtils.getWebApplicationContext(this.getServletContext());
        WebApplicationContext wac = null;
        //已存在WebApplicationContext的情况，同样会跳转至configureAndRefreshWebApplicationContext(wac)
        if (this.webApplicationContext != null) { ... }
        ...
        
        //创建WebApplicationContext
        if (wac == null) { wac = this.createWebApplicationContext(rootContext); }

        //若刷新WebApplicationContext未成功，则再次进行尝试
        if (!this.refreshEventReceived) {
            synchronized(this.onRefreshMonitor) { this.onRefresh(wac); }
        }
        ...
    }
	
    protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
        //contextClass的默认值为XmlWebApplicationContext.class(构造函数赋值)
        Class<?> contextClass = this.getContextClass();
        if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
            ...
        } else {
            //创建WebApplicationContext
            ConfigurableWebApplicationContext wac = (ConfigurableWebApplicationContext)BeanUtils.instantiateClass(contextClass);
            ...
            
            //获取web.xml中配置的contextConfigLocation属性
            String configLocation = this.getContextConfigLocation();
            if (configLocation != null) { wac.setConfigLocation(configLocation); }

            this.configureAndRefreshWebApplicationContext(wac);
            return wac;
        }
    }

    protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
        ...
        //添加监听器ContextRefreshListener，该监听器在容器刷新完成后触发
        wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));
        ...
        
        //WebApplicationContext刷新，结束时触发ContextRefreshListener
        wac.refresh();
    }
    ...
}
```
在springmvc中，提供了ApplicationEventPublisher#publishEvent(Object)（事件发布器）、ApplicationEvent（事件）与 ApplicationListener（事件监听器）。当springmvc通过ApplicationEventPublisher#publishEvent(Object)发布ApplicationEvent（事件）时，ApplicationListener（事件监听器）将会监听到。  
```java
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {
    void refresh() throws BeansException, IllegalStateException;
    ...
}
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext {
    public void refresh() throws BeansException, IllegalStateException {
        synchronized(this.startupShutdownMonitor) {
            ...
            try {
                ...
                //结束WebApplicationContext刷新
                this.finishRefresh();
            } catch (BeansException var10) {
                ...
            } finally { ... }
        }
    }
    protected void finishRefresh() {
        //发送WebApplicationContext刷新结束事件
        this.publishEvent((ApplicationEvent)(new ContextRefreshedEvent(this)));
        ...
    }
    ...
}

public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
    private class ContextRefreshListener implements ApplicationListener<ContextRefreshedEvent> {
        ...
        //WebApplicationContext刷新结束事件触发该监听器
        public void onApplicationEvent(ContextRefreshedEvent event) { FrameworkServlet.this.onApplicationEvent(event); }
    }
    public void onApplicationEvent(ContextRefreshedEvent event) {
        this.refreshEventReceived = true;
        synchronized(this.onRefreshMonitor) { this.onRefresh(event.getApplicationContext()); }
    }
    protected void onRefresh(ApplicationContext context) {}
    ...
}

public class DispatcherServlet extends FrameworkServlet {
    protected void onRefresh(ApplicationContext context) {
        this.initStrategies(context);
    }
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
    ...
}
```

----------------------------------------  

```java
public class DispatcherServlet extends FrameworkServlet {
    private void initHandlerMappings(ApplicationContext context) {
        this.handlerMappings = null;
        //springmvc会加载所有实现了HandlerMapping接口的bean，并进行排序。
        if (this.detectAllHandlerMappings) {
            //配置文件中的<mvc:annotation-driven></mvc:annotation-driven>标签，会创建RequestMappingHandlerMapping的bean，并在此时被扫描到
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
	
        //如果仍未加载HandlerMapping，springmvc会加载默认提供的HandlerMapping
        if (this.handlerMappings == null) {
            this.handlerMappings = this.getDefaultStrategies(context, HandlerMapping.class);
            ...
        }
        ...
    }
    ...
}
```

DispatcherServlet.properties中提供的HandlerMapping默认实现类有三个：  
```
org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping
```

通过<mvc:annotation-driven></mvc:annotation-driven>创建的RequestMappingHandlerMapping，其继承关系：  
RequestMappingHandlerMapping --> RequestMappingInfoHandlerMapping --> AbstractHandlerMethodMapping --> InitializingBean  
在spring创建并初始化bean的过程中，会调用bean中的afterPropertiesSet()方法(实现自InitializingBean接口)。 

```java
public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping implements MatchableHandlerMapping, EmbeddedValueResolverAware {
    public void afterPropertiesSet() {
        ...
        super.afterPropertiesSet();
    }
    
    protected boolean isHandler(Class<?> beanType) {
        //如果含有@Controller或者@RequestMapping的注解，则返回true
        //由于Controller.class添加了@Component注解(@Component表示这是一个bean)，而RequestMapping.class没有，因此两者使用方法并不一样
        //使用@Controller注解，或使用@Component+@RequestMapping注解，都能被识别为控制类
        return AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) || AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class);
    }
    ...
}

public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
    public void afterPropertiesSet() {
        this.initHandlerMethods();
    }

    protected void initHandlerMethods() {
        //获取所有的bean
        String[] var1 = this.getCandidateBeanNames();
        int var2 = var1.length;
        for(int var3 = 0; var3 < var2; ++var3) {
            String beanName = var1[var3];
            if (!beanName.startsWith("scopedTarget.")) {
                //解析bean
                this.processCandidateBean(beanName);
            }
        }
        ...
    }

    protected void processCandidateBean(String beanName) {
        Class<?> beanType = null;
        try {
            //获取bean的类型
            beanType = this.obtainApplicationContext().getType(beanName);
        } catch (Throwable var4) { ... }
        //如果bean的类型是handler，则进一步解析
        if (beanType != null && this.isHandler(beanType)) {
            this.detectHandlerMethods(beanName);
        }

    }
    
    //判断bean类型的方法，实际由子类RequestMappingHandlerMapping实现
    protected abstract boolean isHandler(Class<?> beanType);
    
    protected void detectHandlerMethods(Object handler) {
        Class<?> handlerType = handler instanceof String ? this.obtainApplicationContext().getType((String)handler) : handler.getClass();
        if (handlerType != null) {
            Class<?> userType = ClassUtils.getUserClass(handlerType);
            
            //获取类中的所有含有@RequestMapping注解的方法
            Map<Method, T> methods = MethodIntrospector.selectMethods(userType, (method) -> {
                try {
                    return this.getMappingForMethod(method, userType);
                } catch (Throwable var4) { ... }
            });
            ...

            //注册所有的RequestMapping
            methods.forEach((method, mapping) -> {
                Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
                this.registerHandlerMethod(handler, invocableMethod, mapping);
            });
        }

    }
    
    //获取类中的所有含有@RequestMapping注解的方法，由子类RequestMappingHandlerMapping实现
    protected abstract T getMappingForMethod(Method method, Class<?> handlerType);
    ...
}
```

BeanNameUrlHandlerMapping的继承关系：
BeanNameUrlHandlerMapping --> AbstractDetectingUrlHandlerMapping --> AbstractUrlHandlerMapping --> AbstractHandlerMapping --> WebApplicationObjectSupport --> ApplicationObjectSupport --> ApplicationContextAware
不同于RequestMappingHandlerMapping，BeanNameUrlHandlerMapping的初始化入口为ApplicationContextAware接口的setApplicationContext(ApplicationContext applicationContext)方法。
