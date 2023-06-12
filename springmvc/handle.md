
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
        this.logRequest(request);
        Map<String, Object> attributesSnapshot = null;
        if (WebUtils.isIncludeRequest(request)) {
            attributesSnapshot = new HashMap();
            Enumeration<?> attrNames = request.getAttributeNames();

            label116:
            while(true) {
                String attrName;
                do {
                    if (!attrNames.hasMoreElements()) {
                        break label116;
                    }

                    attrName = (String)attrNames.nextElement();
                } while(!this.cleanupAfterInclude && !attrName.startsWith("org.springframework.web.servlet"));

                attributesSnapshot.put(attrName, request.getAttribute(attrName));
            }
        }

        request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.getWebApplicationContext());
        request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
        request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
        request.setAttribute(THEME_SOURCE_ATTRIBUTE, this.getThemeSource());
        if (this.flashMapManager != null) {
            FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
            if (inputFlashMap != null) {
                request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
            }

            request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
            request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
        }

        RequestPath previousRequestPath = null;
        if (this.parseRequestPath) {
            previousRequestPath = (RequestPath)request.getAttribute(ServletRequestPathUtils.PATH_ATTRIBUTE);
            ServletRequestPathUtils.parseAndCache(request);
        }

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
}
```
