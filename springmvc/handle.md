
DispatcherServlet处理请求的入口为void service(HttpServletRequest req, HttpServletResponse resp)方法，但DispatcherServlet并未重写该方法，所以实际调用的是其父类FrameworkServlet中的方法：
```java
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
        if (httpMethod != HttpMethod.PATCH && httpMethod != null) {
            
            //请求正确的情况下，虽然跳转到了父类，但只是根据请求方式调用this.doGet、this.doPost等方法，而这些方法又被FrameworkServlet重写，所以还是跳转到FrameworkServlet中的doGet、doPost等方法
            //而FrameworkServlet中的doGet、doPost等方法，只是简单的调用了this.processRequest(request, response)
            super.service(request, response);
        } else {
            this.processRequest(request, response);
        }
    }
    
    protected final void processRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        ...
        try {
            this.doService(request, response);
        } catch (IOException | ServletException var16) {
            ...
        } catch (Throwable var17) {
            ...
        } finally { ... }
    }
    
    protected abstract void doService(HttpServletRequest request, HttpServletResponse response) throws Exception;
    ...
}

public class DispatcherServlet extends FrameworkServlet {
    protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ...
        try {
            //配置完一些属性后，进一步跳转
            this.doDispatch(request, response);
        } finally { ... }
    }
    
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ...
        try {
            try {
                ...
                try {
                    //判断是否为文件上传
                    processedRequest = this.checkMultipart(request);
                    
                    multipartRequestParsed = processedRequest != request;
                    //获取能处理该请求的handler
                    mappedHandler = this.getHandler(processedRequest);
                    ...

                    //适配器模式，mappedHandler.getHandler()返回值是Object，而handler有可能定义在类或者方法上，这种设计模式是为了解决实际的多态转换
                    HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());
                    ...

                    //执行预处理(拦截器)
                    if (!mappedHandler.applyPreHandle(processedRequest, response)) { return; }

                    //执行handler方法
                    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                    ...

                    //执行后处理(拦截器)
                    this.applyDefaultViewName(processedRequest, mv);
                    mappedHandler.applyPostHandle(processedRequest, response, mv);
                } catch (Exception var20) {
                    ...
                } catch (Throwable var21) {
                    ...
                }
                this.processDispatchResult(processedRequest, response, mappedHandler, mv, (Exception)dispatchException);
            } catch (Exception var22) {
                ...
            } catch (Throwable var23) {
                ...
            }

        } finally {
            if (asyncManager.isConcurrentHandlingStarted()) {
                if (mappedHandler != null) {
                    mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
                }
            } else if (multipartRequestParsed) {
                this.cleanupMultipart(processedRequest);
            }

        }
    }
    
    protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        if (this.handlerMappings != null) {
            //获取所有的HandlerMapping
            Iterator var2 = this.handlerMappings.iterator();
            
            while(var2.hasNext()) {
                HandlerMapping mapping = (HandlerMapping)var2.next();
                
                //根据HandlerMapping获取handler，比如RequestMappingHandlerMapping
                HandlerExecutionChain handler = mapping.getHandler(request);
                if (handler != null) { return handler; }
            }
        }
        return null;
    }
    ...
}
```
以RequestMappingHandlerMapping的getHandler为例：
```java
public abstract class AbstractHandlerMapping extends WebApplicationObjectSupport implements HandlerMapping, Ordered, BeanNameAware {
    public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        Object handler = this.getHandlerInternal(request);
        ...
    }
    protected abstract Object getHandlerInternal(HttpServletRequest request) throws Exception;
    ...
}

public abstract class RequestMappingInfoHandlerMapping extends AbstractHandlerMethodMapping<RequestMappingInfo> {
    protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
        ...
        try {
            var2 = super.getHandlerInternal(request);
        } finally {
            ...
        }
        return var2;
    }
    ...
}

public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
    protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
        //获取请求路径
        String lookupPath = this.initLookupPath(request);
        ...
        
        try {
            //根据路径查找handler
            HandlerMethod handlerMethod = this.lookupHandlerMethod(lookupPath, request);
            var4 = handlerMethod != null ? handlerMethod.createWithResolvedBean() : null;
        } finally {
            this.mappingRegistry.releaseReadLock();
        }
        return var4;
    }
    protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
        List<AbstractHandlerMethodMapping<T>.Match> matches = new ArrayList();
        //从mappingRegistry中查找路径对应的handler
        List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
        ...
    }
    public List<T> getMappingsByDirectPath(String urlPath) {
        //map集合的get方法
        return (List)this.pathLookup.get(urlPath);
    }
    ...
}
```
通过适配器模式获取到对应的HandlerAdapter后，会通过HandlerAdapter的handle方法，来执行我们所编写的处理请求的方法：
```java
public abstract class AbstractHandlerMethodAdapter extends WebContentGenerator implements HandlerAdapter, Ordered {
    public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        return this.handleInternal(request, response, (HandlerMethod)handler);
    }
    protected abstract ModelAndView handleInternal(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception;
}

public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter implements BeanFactoryAware, InitializingBean {
    protected ModelAndView handleInternal(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        this.checkRequest(request);
        ModelAndView mav;
        
        //是否对多个请求同步Session
        if (this.synchronizeOnSession) {
            HttpSession session = request.getSession(false);
            if (session != null) {
                Object mutex = WebUtils.getSessionMutex(session);
                synchronized(mutex) {
                    mav = this.invokeHandlerMethod(request, response, handlerMethod);
                }
            } else {
                mav = this.invokeHandlerMethod(request, response, handlerMethod);
            }
        } else {
            mav = this.invokeHandlerMethod(request, response, handlerMethod);
        }

        if (!response.containsHeader("Cache-Control")) {
            if (this.getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
                this.applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
            } else {
                this.prepareResponse(response);
            }
        }

        return mav;
    }
    protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        ServletWebRequest webRequest = new ServletWebRequest(request, response);

        ModelAndView var15;
        try {
            WebDataBinderFactory binderFactory = this.getDataBinderFactory(handlerMethod);
            ModelFactory modelFactory = this.getModelFactory(handlerMethod, binderFactory);
            ServletInvocableHandlerMethod invocableMethod = this.createInvocableHandlerMethod(handlerMethod);
            if (this.argumentResolvers != null) {
                invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
            }

            if (this.returnValueHandlers != null) {
                invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
            }

            invocableMethod.setDataBinderFactory(binderFactory);
            invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
            ModelAndViewContainer mavContainer = new ModelAndViewContainer();
            mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
            modelFactory.initModel(webRequest, mavContainer, invocableMethod);
            mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
            AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
            asyncWebRequest.setTimeout(this.asyncRequestTimeout);
            WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
            asyncManager.setTaskExecutor(this.taskExecutor);
            asyncManager.setAsyncWebRequest(asyncWebRequest);
            asyncManager.registerCallableInterceptors(this.callableInterceptors);
            asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);
            Object result;
            if (asyncManager.hasConcurrentResult()) {
                result = asyncManager.getConcurrentResult();
                mavContainer = (ModelAndViewContainer)asyncManager.getConcurrentResultContext()[0];
                asyncManager.clearConcurrentResult();
                LogFormatUtils.traceDebug(this.logger, (traceOn) -> {
                    String formatted = LogFormatUtils.formatValue(result, !traceOn);
                    return "Resume with async result [" + formatted + "]";
                });
                invocableMethod = invocableMethod.wrapConcurrentResult(result);
            }

            invocableMethod.invokeAndHandle(webRequest, mavContainer, new Object[0]);
            if (asyncManager.isConcurrentHandlingStarted()) {
                result = null;
                return (ModelAndView)result;
            }

            var15 = this.getModelAndView(mavContainer, modelFactory, webRequest);
        } finally {
            webRequest.requestCompleted();
        }

        return var15;
    }
}
```
