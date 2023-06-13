
DispatcherServlet处理请求的入口为void service(HttpServletRequest req, HttpServletResponse resp)方法，但DispatcherServlet并未重写该方法，所以实际调用的是其父类FrameworkServlet中的方法：
```java {.line-numbers}
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
        //进行一些配置后，进一步跳转
        ...
        try {
            this.doDispatch(request, response);
        } finally {
            if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted() && attributesSnapshot != null) {
                this.restoreAttributesAfterInclude(request, attributesSnapshot);
            }
            if (this.parseRequestPath) {
                ServletRequestPathUtils.setParsedRequestPath(previousRequestPath, request);
            }
        }
    }
    
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ...
        try {
            try {
                ModelAndView mv = null;
                Exception dispatchException = null;
                try {
                    processedRequest = this.checkMultipart(request);
                    multipartRequestParsed = processedRequest != request;
                    //获取能处理该请求的handler
                    mappedHandler = this.getHandler(processedRequest);
                    if (mappedHandler == null) {
                        this.noHandlerFound(processedRequest, response);
                        return;
                    }

                    HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());
                    String method = request.getMethod();
                    boolean isGet = HttpMethod.GET.matches(method);
                    if (isGet || HttpMethod.HEAD.matches(method)) {
                        long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                        if ((new ServletWebRequest(request, response)).checkNotModified(lastModified) && isGet) {
                            return;
                        }
                    }

                    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                        return;
                    }

                    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                    if (asyncManager.isConcurrentHandlingStarted()) {
                        return;
                    }

                    this.applyDefaultViewName(processedRequest, mv);
                    mappedHandler.applyPostHandle(processedRequest, response, mv);
                } catch (Exception var20) {
                    dispatchException = var20;
                } catch (Throwable var21) {
                    dispatchException = new NestedServletException("Handler dispatch failed", var21);
                }

                this.processDispatchResult(processedRequest, response, mappedHandler, mv, (Exception)dispatchException);
            } catch (Exception var22) {
                this.triggerAfterCompletion(processedRequest, response, mappedHandler, var22);
            } catch (Throwable var23) {
                this.triggerAfterCompletion(processedRequest, response, mappedHandler, new NestedServletException("Handler processing failed", var23));
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

    
}
```
