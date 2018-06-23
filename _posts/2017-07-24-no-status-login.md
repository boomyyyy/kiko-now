---
layout: post
title: "Cookie & ThreadLocal 代替 session 实现无状态登录"
description: "cookie & ThreadLocal 存取当前用户状态"
date: 2017-07-24
tags: [Java]
comments: true
share: true
---

> session 本质来说就是服务器中一块内存，在单点系统中使用 session 没有什么问题，在集群环境中使用了负载均衡，如果用户在 A 服务器登录，而后将该用户的请求负载均衡到 B 服务器，就会出现将用户踢到登录页，导致用户体验很差。那么替代方案来了，那就是使用  cookie 和 ThreadLocal 来替代 session。代码如下~  

```java
/**
* 当前用户类
*/
public class Principal {
	private int id;
	private String username;
	private String password;
	private String token;

	// 省略 getter setter
}
```

```java
/**
* 描述类
*/
public class PrincipalHolder {

	private PrincipalHolder {
	}

	private static final ThreadLocal<Principal> PRINCIPAL = new ThreadLocal<>();

	/**
	* 设置当前用户
	*/
	public static void set(Principal principal){
		PRINCIPAL.set(principal);
	}

	/**
	* 获取当前用户
	*/
	public static Principal get(){
		PRINCIPAL.get();
	}

}
```

```java
/**
* 拦截器，拦截请求，通过 cookie 中的 token 获取当前用户信息，并放到 ThreadLocal 中
*/
public class PrincipalFilter implements Filter{

	@Override
	public void init(FilterConfig filterConfig) throws ServletException {
	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
	 								throws IOException, ServletException {
		HttpServletRequest req = (HttpServletRequest) request;
		Cookie[] cookies = req.getCookies();
		for (Cookie cookie : cookies) {
			if (cookie.getName().equals("token")) {
				// 通过 token 获取用户信息
				Principal principal = SpringUtil.getBean(UserService.class).getByToken(cookie.getValue());
				PrincipalHolder.set(principal);
				break;
			}
		}
		chain.doFilter(request, response);
	}

	@Override
	public void destroy() {

	}

}
```

```java
/**
* 简单的登录逻辑
*/
@PostMapping("/login")
public HttpEntity<Principal> login(@RequestBody LoginParam loginParam, HttpServletResponse response) {
	Principal principal;
	try {
		principal = userService.login(loginParam);
	} catch (Exception e) {
		Throwables.getStackTraceAsString(e);
	}
	Cookie cookie = new Cookie("token", principal.getToken());
	cookie.setDomain("domain.com");
	cookie.setPath("/");
	response.addCookie(cookie);
	return new ResponseEntity<>(principal, HttpStatus.OK);
}
```

```java
/**
* SpringBoot 配置 filter 的方式，普通的 web 项目在 web.xml 中配置即可
*/
@Configuration
public class FilterConfig {

	@Bean
	public Filter principalFilter() {
		return new PrincipalFilter();
	}

	@Bean
	public FilterRegistrationBean principalFilterRegistration() {
		FilterRegistrationBean registration = new FilterRegistrationBean();
		registration.setFilter(principalFilter());
		registration.addUrlPatterns("/*");
		registration.setName("principalFilter");
		registration.setOrder(1);
		return registration;
	}

}
```

```java
/**
* 测试获取当前用户
*/
@GetMapping("/principal")
public HttpEntity<Principal> getPrincipal(){
	return new ResponseEntity<>(PrincipalHolder.get(), HttpStatus.OK);
}
```
