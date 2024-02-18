---
title: spring-security中遇见的耗时小坑
date: 2023-04-10 22:01:52
categories:
- code
tags: 
- code
- spring-security
- cas
- relogin
---

## :pear: 背景

客户现场运维同事反馈某系统输入正确的用户名、密码后，无法进入系统首页。地址栏中地址却在 SSO server 和系统地址之间来回跳转，系统日志中也没有相关的日志提供线索。听到这里就晓得，不是一个运维同学在白盒的情况下，能解决的问题了。

## :orange: 问题跟进

### :one: 明确运维说『来回跳转』到底涉及那些地址

让运维同学 F12 打开 chrome 浏览器开发者模式，切换 Network 页签勾选 `Preserve log`（保留请求日志），就能记录浏览器『来回跳转』网络请求。

```
${应用}/login/cas?st=xxxxx
${SSO-server}/login?service=encodeURIComponent(${应用}/login/cas)
```

就是这两个地址『来回跳转』。从这里能得做出如下推断:

1. SSO server 服务依据 cookie 中的 TGT 验证是通过的。否则，此处展示的就是登录页面。
2. 应用 `/login/cas` 请求，通过后台调用 SSO server 的 validate 接口完成 st（`service ticket`）验证，显然这里是不通过的。否则，此处展示的就是应用首页面。

总结，说明问题出在应用 `/login/cas` 请求认证中，虽然不敢相信但『事实胜于雄辩』。

### :two: 在应用后端找找原因

为什么上面说 `/login/cas` 不敢相信有问题，因为应用后端使用 `spring-security-cas` 组件，而这个组件怕是有成千上万的项目使用已经是很优秀的组件，难道被我碰见开源的 BUG 了？

`spring-security-cas` 组件，有几个重要组成类：

- `CasAuthenticationFilter`：`CAS` 验证过滤器，`/login/cas` 请求验证入口。其中， `attemptAuthentication()` 其中包含验证方法 ，验证不通过则抛出 `AuthenticationException`。`successfulAuthentication` 则是验证通过后执行逻辑，可以是重定向到首页，或是继续访问后续逻辑。`unsuccessfulAuthentication` 则是验证失败后执行逻辑，是重定向到 `/login` 登录请求。
- `CasAuthenticationProvider`：真正的 CAS 验证入口，主要完成 CAS 验证和用户权限信息组装。
- `AbstractCasProtocolUrlBasedTicketValidator`：被 `CasAuthenticationProvider` 类调用，完成调用 SSO server validate 接口验证`serviceTicket`。
- `AbstractCasAssertionUserDetailsService`：被 `CasAuthenticationProvider` 类调用，完成用户及权限信息的装载。

关键代码如下，其中会导致进入 `unsuccessfulAuthentication()` 逻辑的，是抛出 `TicketValidationException` 异常。那也就是 `ticketValidator.validate()` 、 `authenticationUserDetailsService.loadUserDetails()` 或 `userDetailsChecker.check()` 逻辑点抛出异常。

```java
// CasAuthenticationProvider
private CasAuthenticationToken authenticateNow(final Authentication authentication) throws AuthenticationException {
  try {
    Assertion assertion = this.ticketValidator.validate(authentication.getCredentials().toString(), getServiceUrl(authentication));
    UserDetails userDetails = loadUserByAssertion(assertion);
    this.userDetailsChecker.check(userDetails);
    return new CasAuthenticationToken(this.key, userDetails, authentication.getCredentials(), this.authoritiesMapper.mapAuthorities(userDetails.getAuthorities()), userDetails, assertion);
  } catch (TicketValidationException ex) {
    throw new BadCredentialsException(ex.getMessage, ex);
  }
}
protected UserDetails loadUserByAssertion(final Assertion assertion) {
  final CasAssertionAuthenticationToken token = new CasAssertionAuthenticationToken(assertion, "");
  return this.authenticationUserDetailsService.loadUserDetails(token);
}
```
而后，现场通过 `arthas` 工具代码 `watch` 命令，监听代码明确异常抛出位置。最后，是在 `authenticationUserDetailsService.loadUserDetails()` 应用自己实现类上抛出了  `UsernameNotFoundException` 异常。

```java
public UserDetails loadUserDetails(CasAssertionAuthenticationToken token) {
        Map<String, Object> attributes = getPrincipalAttributes(token);
        if (MapUtils.isEmpty(attributes)) {
            log.warn("{} 没有额外的元数据", token.getName());
            throw new UsernameNotFoundException(token.getName() + "没有额外的元数据");
        }
        String loginId = getLoginId(attributes);
        if (StringUtils.isBlank(loginId)) {
            log.warn("{} 没有登录标识", loginId);
            throw new UsernameNotFoundException(String.format("%s没有登录标识", loginId));
        }
        String corpId = getCorpId(attributes);
        UserDO userDO = loadUserDO(loginId, corpId);
        List<GrantedAuthority> grantedAuthorities = getDefaultUserAuthorities(userDO.getId());
        return new CommonUserDetails(userDO.getLoginId(), userDO.getPassword(), grantedAuthorities, userDO);
    }
```

### :three: 到底是为什么有异常没有日志输出呢？

最终，还是要回到文首『破』代码有异常，日志文件却无记录问题。回头细看 `AbstractAuthenticationProcessingFilter.unsuccessfulAuthentication()` 方法，异常日志打印居然是  trace 级别，现场日志级别配置的 error 级别，故代码有异常，日志文件却无记录问题。

```java
protected void unsuccessfulAuthentication(HttpServletRequest request,
			HttpServletResponse response, AuthenticationException failed)
			throws IOException, ServletException {
		SecurityContextHolder.clearContext();
			logger.trace("Failed to process authentication request", failed);
			logger.trace("Cleared SecurityContextHolder");
			logger.trace("Handling authentication failure");
		rememberMeServices.loginFail(request, response);
		failureHandler.onAuthenticationFailure(request, response, failed);
	}
```

### :banana: 未提及的基础知识

- [CAS 官网](https://apereo.github.io/cas/6.6.x/index.html)
