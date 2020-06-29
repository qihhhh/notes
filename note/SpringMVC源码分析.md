## 1. DispatchServlet

* 架构：

<img src="C:\Users\qihang\AppData\Local\Temp\Image.png" alt="Image" style="zoom: 80%;" />

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HttpServletRequest processedRequest = request;
        HandlerExecutionChain mappedHandler = null;
        boolean multipartRequestParsed = false;
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        try {
            ModelAndView mv = null;
            Exception dispatchException = null;
            try {
               //1、检查是否文件上传请求
                processedRequest = checkMultipart(request);
                multipartRequestParsed = processedRequest != request;
                // Determine handler for the current request.
               //*2、根据当前的请求地址找到那个类能来处理；
                mappedHandler = getHandler(processedRequest);
               //3、如果没有找到哪个处理器（控制器）能处理这个请求就404，或者抛异常
                if (mappedHandler == null || mappedHandler.getHandler() == null) {
                    noHandlerFound(processedRequest, response);
                    return;
                }
                // Determine handler adapter for the current request.
               //*4、拿到能执行这个类的所有方法的适配器；（反射工具AnnotationMethodHandlerAdapter）
                HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
                // Process last-modified header, if supported by the handler.
                String method = request.getMethod();
                boolean isGet = "GET".equals(method);
                if (isGet || "HEAD".equals(method)) {
                    long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                    if (logger.isDebugEnabled()) {
                        String requestUri = urlPathHelper.getRequestUri(request);
                        logger.debug("Last-Modified value for [" + requestUri + "] is: " + lastModified);
                    }
                    if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                        return;
                    }
                }
                if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }
                try {
                    // Actually invoke the handler.处理（控制）器的方法被调用
                    //控制器（Controller），处理器（Handler）
                    //*5、适配器来执行目标方法；将目标方法执行完成后的返回值作为视图名，设置保存到ModelAndView中
                    //目标方法无论怎么写，最终适配器执行完成以后都会将执行后的信息封装成ModelAndView
                    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                }
                finally {
                    if (asyncManager.isConcurrentHandlingStarted()) {
                        return;
                    }
                }
                applyDefaultViewName(request, mv);//如果没有视图名设置一个默认的视图名；
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            }
            catch (Exception ex) {
                dispatchException = ex;
            }
            //转发到目标页面；
            //*6、根据方法最终执行完成后封装的ModelAndView；转发到对应页面，而且ModelAndView中的数据可以从请求域中获取
            processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
        }
        catch (Exception ex) {
            triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
        }
        catch (Error err) {
            triggerAfterCompletionWithError(processedRequest, response, mappedHandler, err);
        }
        finally {
            if (asyncManager.isConcurrentHandlingStarted()) {
                // Instead of postHandle and afterCompletion
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
                return;
            }
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
```

* 整体流程：

![img](https://uploadfiles.nowcoder.com/images/20191203/5459305_1575381897289_4A47A0DB6E60853DEDFCFDF08A5CA249)

1)所有请求过来DispatcherServlet收到请求

2)调用doDispatch()方法进行处理

​		2.1) getHandler()：根据当前请求地址找到能处理这个请求的目标处理器类（处理器）：根据当前请求在HandlerMapping中找到这个请求的映射信息，获取到目标处理器类。

​		2.2）getHandlerAdapter()：根据当前处理器类获取到能执行这个处理器方法的适配器：根据当前处理器类，找到当前类的HandlerAdapter（适配器）

​		2.3）使用刚才获取到的适配器（AnnotationMethodHandlerAdapter ）执行目标方法;

​		2.4）目标方法执行后会返回一个ModelAndView对象

​		2.5）根据ModelAndView的信息转发到具体的页面，并可以在请求域中取出ModelAndView中的模型数据

***



### 细节一：getHandler()

```java
mappedHandler = getHandler(processedRequest);
```

![image-20200607134143967](C:\Users\qihang\AppData\Roaming\Typora\typora-user-images\image-20200607134143967.png)

* 说明：getHandler()会返回目标处理器类的执行链

***



```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		for (HandlerMapping hm : this.handlerMappings) {
			if (logger.isTraceEnabled()) {
				logger.trace(
						"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
			}
			HandlerExecutionChain handler = hm.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
		return null;
	}
```

![image-20200607134803327](C:\Users\qihang\AppData\Roaming\Typora\typora-user-images\image-20200607134803327.png)

* 说明：HandlerMapping：处理器映射：他里面保存了每一个处理器能处理哪些请求的映射信息；



### 细节二：getHandlerAdapter()

```java
 // Determine handler adapter for the current request.
 HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		for (HandlerAdapter ha : this.handlerAdapters) {
			if (logger.isTraceEnabled()) {
				logger.trace("Testing handler adapter [" + ha + "]");
			}
			if (ha.supports(handler)) {
				return ha;
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```

![image-20200607135731908](C:\Users\qihang\AppData\Roaming\Typora\typora-user-images\image-20200607135731908.png)

### 细节三：handle()--执行目标方法

```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

* 断点调试：

```java
public String updateBook(@RequestParam(value="author")String author,Map<String, Object> model,                      HttpServletRequest request, @ModelAttribute("haha")Book book

            )

mv = ha.handle(processedRequest, response, mappedHandler.getHandler());执行目标方法的细节；
     ||
     \/

return invokeHandlerMethod(request, response, handler);
     ||
     \/
protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, Object handler)throws Exception {

         //拿到方法的解析器
        ServletHandlerMethodResolver methodResolver = getMethodResolver(handler);
          //方法解析器根据当前请求地址找到真正的目标方法
        Method handlerMethod = methodResolver.resolveHandlerMethod(request);
          //创建一个方法执行器；
        ServletHandlerMethodInvoker methodInvoker = new ServletHandlerMethodInvoker(methodResolver);
          //包装原生的request, response，
        ServletWebRequest webRequest = new ServletWebRequest(request, response);
          //创建了一个，隐含模型
        ExtendedModelMap implicitModel = new BindingAwareModelMap();
 //真正执行目标方法；目标方法利用反射执行期间确定参数值，提前执行modelattribute等所有的操作都在这个方法中；
        Object result = methodInvoker.invokeHandlerMethod(handlerMethod, handler, webRequest, implicitModel);
         
        ModelAndView mav =
                methodInvoker.getModelAndView(handlerMethod, handler.getClass(), result, implicitModel, webRequest);
        methodInvoker.updateModelAttributes(handler, (mav != null ? mav.getModel() : null), implicitModel, webRequest);
        return mav;
    }
```

* 接上：invokeHandlerMethod执行细节

```java

```



## 2. SpringMVC九大组件

> SpringMVC在工作的时候，关键位置都是由这些组件完成的；
>
> 共同点：九大组件全部都是接口；接口就是规范；提供了非常强大的扩展性；

```java
	 /** 文件上传解析器*/
    private MultipartResolver multipartResolver;

    /** 区域信息解析器；和国际化有关 */
    private LocaleResolver localeResolver;

    /** 主题解析器；强大的主题效果更换 */
    private ThemeResolver themeResolver;

    /** Handler映射信息；HandlerMapping */
    private List<HandlerMapping> handlerMappings;

    /** Handler的适配器 */
    private List<HandlerAdapter> handlerAdapters;

    /** SpringMVC强大的异常解析功能；异常解析器 */
    private List<HandlerExceptionResolver> handlerExceptionResolvers;

    /**  */
    private RequestToViewNameTranslator viewNameTranslator;

    /** FlashMap+Manager：SpringMVC中运行重定向携带数据的功能 */
    private FlashMapManager flashMapManager;

    /** 视图解析器； */
    private List<ViewResolver> viewResolvers;
```

### 组件的初始化细节

```java
/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
//在DispathServlet中：
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);//
		initHandlerAdapters(context);//
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}

```

* 举例：初始化HandlerMappings:

```java
/**
	 * Initialize the HandlerMappings used by this class.
	 * <p>If no HandlerMapping beans are defined in the BeanFactory for this namespace,
	 * we default to BeanNameUrlHandlerMapping.
	 */
	private void initHandlerMappings(ApplicationContext context) {
		this.handlerMappings = null;

		if (this.detectAllHandlerMappings) {
			// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
			Map<String, HandlerMapping> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
				// We keep HandlerMappings in sorted order.
				OrderComparator.sort(this.handlerMappings);
			}
		}
		else {
			try {
				HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
				this.handlerMappings = Collections.singletonList(hm);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerMapping later.
			}
		}

		// Ensure we have at least one HandlerMapping, by registering
		// a default HandlerMapping if no other mappings are found.
		if (this.handlerMappings == null) {
			this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
			}
		}
	}
```





