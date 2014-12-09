---
layout: post
title: "Spring mvc的controller传递HttpServletResponse参数的一点事"
date: 2014-12-09 10:05:41 +0800
comments: true
categories: 
---

<pre>
@RequestMapping(value = "cardDown", method = RequestMethod.GET, headers = "Accept=text/html")
    public void cardDown(ModelMap modelMap, HttpServletRequest request, HttpServletResponse response, String id, int status){
    ......
    }
</pre>

之前在使用Spring mvc的时候发现这么一回事：在spring mvc的controller的参数里如果有HttpServletResponse(类似上面的代码),那么必须有返回值框架才会去在执行完handler后去搜索相应的viewResolver和view从而展现数据。如果没有返回值，那么默认就是返回null的。我初步推测框架的处理过程大致如此：如果controller参数里传递HttpServletResposne的话，框架就认为视图由handler自己生成可以不参于这个过程,但是如果handler有返回值的话，那么仍然认为还需要参与到视图生成的过程中。

<!--more-->

翻了一下spring mvc的代码，验证了自己的想法。在DispatchServlet的921行
<pre>
// Actually invoke the handler.
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
</pre>
这里的mv就是视图生成的结果。接着经历下面的过程：

1. AbstractHandlerMethodAdapter.handle
2. RequestMappingHandlerAdapter.handleInternal
3. RequestMappingHandlerAdapter.invokeHandleMethod
	
	这地方有一个关键的变量mavContainer
	<pre>
	ModelAndViewContainer mavContainer = new ModelAndViewContainer();
	</pre>
	mavContainer有一个属性requestHandled，其标志着此次请求是否是由handler自己控制的。默认为false。
	
4. ServletInvocableHandlerMethod.invokeAndHandle
5. InvocableHandlerMethod.invokeForRequest
6. InvocableHandlerMethod.getMethodArgumentValues

 	这个方法的功能在名字上大体就能看出来：获取controller中每一个参数的值。关键的地方在于
 	<pre>
 	args[i] = argumentResolvers.resolveArgument(parameter, mavContainer, request, dataBinderFactory);
 	</pre>
 	这一行代码关联的是对每一中paramerter的处理类。接下来的调用见7
7. HandlerMethodArgumentResolverComposite

	<pre>
	HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
		Assert.notNull(resolver, "Unknown parameter type [" + parameter.getParameterType().getName() + "]");
		return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
	</pre>
	
	这里的代码就三行，第一步是根据参数不同，获取不同的argumentResolver。当参数为HttpServletResponse的时候，就会调用ServletResponseMethodArgumentResolver.resolveArgument
	
8. ServletResponseMethodArgumentResolver.resolveArgument

	最核心的一段代码来了
	
	<pre>
	if (mavContainer != null) {
		mavContainer.setRequestHandled(true);
	}
	</pre>
	
	这里就把mavContainer的requestHandled设置为了true.

9. 回到4，调用完InvocableHandlerMethod.invokeForRequest

	<pre>
	if (returnValue == null) {
		if (isRequestNotModified(webRequest) || hasResponseStatus() || mavContainer.isRequestHandled()) {
			mavContainer.setRequestHandled(true);
			return;
		}
	}else if (StringUtils.hasText(this.responseReason)) {
		mavContainer.setRequestHandled(true);
		return;
	}
	
	mavContainer.setRequestHandled(false);
	</pre>
	
	当handler的返回值为null的时候，直接返回。否则将mavContainer的requestHandled设置为false。

10. 接着回到3，调用完ServletInvocableHandlerMethod.invokeAndHandle后，接着调用getModelAndView(mavContainer, modelFactory, webRequest)

11. ServletInvocableHandlerMethod.getModelAndView

	<pre>
	modelFactory.updateModel(webRequest, mavContainer);

	if (mavContainer.isRequestHandled()) {
		return null;
	}
	</pre>
	
	这里当mavContainer的requestHandled被设置为true时,视图返回null。
	
整个大体的流程就是这样的，如果需要使用HttpServletResponse同时还需要框架控制视图生成的话，可以给controller method一个返回值（随便什么都行）。
	





