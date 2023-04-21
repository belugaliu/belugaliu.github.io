---
title: spring-boot-security OAuth2应用实现
date: 2023-04-11 00:08:54
categories:
- spring-security
  tags:
- spring-security
- cas
- oauth2
- pre-security
---

## :mouse: 背景

项目上一直使用 CAS + 应用 session + nginx IP hash 组合方式实现伪集群部署。但这种方式也有一定的缺点，请求不够平均，应用使用异步处理方式，还必须将结果返回给发起的应用，否则前端无法拿到结果。这些都是 IP 绑定固定应用导致的。2023 年为止，在网上搜索到主要解决方式有两个: 1. session 共享（需要依赖 redis）2. 签发 JWT 授权。不想引入 redis，所以选择签发 JWT 授权。技术选型上使用 `spring-scurity` + `CAS` + `oauth2` 组合方式。

## :tiger: 应用实现

使用 `spring-security` + `CAS` + `Oauth2` 组合方式，spring-boot 中提供了很多 starter 可以使用。以下使用 maven 仓库管理为例

```xml
<!-- spring-security 基础 -->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
<!-- CAS 相关 -->
<dependency>
	<groupId>org.springframework.security</groupId>
	<artifactId>spring-security-cas</artifactId>
</dependency>
<!-- security-jwt相关 -->
<dependency>
	<groupId>org.springframework.security</groupId>
	<artifactId>spring-security-oauth2-jose</artifactId>
</dependency>
<!-- JWT 签发 -->
<dependency>
	<groupId>org.springframework.security</groupId>
	<artifactId>spring-security-oauth2-resource-server</artifactId>
</dependency>
```

- `spring-boot-starter-security`，是 `spring-security` 的基础包，主要包含 `spring-security-config` 和 `spring-security-web`
- `spring-security-cas`，是 `spring-security` 的 CAS 相关，包含 CAS validation 及验证通过或不通过的处理
- `spring-security-oauth2-jose`，是 `spring-security` 的 token 验证相关
- `spring-security-oauth2-resource-server`，是 `spring-security` 的 token 签发及 web token 验证。

### :elephant: 怎么集成

集成之前，建议先看看[ spring-security 的架构](https://docs.spring.io/spring-security/reference/servlet/architecture.html)，:smile_cat: 很容易理解。当然集成一个组件，需要有集成思路或步骤，

#### PART1. 选定用户校验方式

spring-security 目前有支持的集中方式，

- 伪验证（`AbstractPreAuthenticatedProcessingFilter`）其中假设委托人已经由外部系统进行了身份验证，实现类完成简单的校验
- CAS 验证（`CasAuthenticationFilter`）
- 本地登录验证（`UsernamePasswordAuthenticationFilter`）应用本地数据库验证，非独立验证服务
- token 验证（`BearerTokenAuthenticationFilter`）OAuth2 JWT 签发的 token 应用服务验证
- 其他验证（`AbstractAuthenticationProcessingFilter`）实现此类来完成定制验证，比如约定好请求授权的验证方式。

当然，此处只是选择 Filter 并非真正验证的位置。所以，spring-security 支持的 CAS 或  UsernamePassword 验证方式也是把你的验证器默认设置了。

#### PART2. 组装待校验元素（principal/credentials）

principal，被验证主体。credentials，被验证证书，也可以是密码。

如果是使用 CAS 或 UsernamePassword 验证，可以跳过这里，因为组装待验证元素，有默认实现。就拿 CAS 来比如

```java
// CASAuthenticationFilter
public Authentication attemptAuthentication(HttpServletRequest request, 
                                            HttpServletResponse response)
			throws AuthenticationException, IOException {
  //,,,
  UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(
      username, password);
  //,,,
}
// UsernamePasswordAuthenticationToken
public class UsernamePasswordAuthenticationToken extends AbstractAuthenticationToken {
  public UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
		//,,,,
		this.principal = principal;
		this.credentials = credentials;
		setAuthenticated(false);
	}
}
```

如果是使用 `AbstractPreAuthenticatedProcessingFilter` 时，则需要覆盖方法 `getPreAuthenticatedPrincipal()` 和 `getPreAuthenticatedPrincipal()` 来确定主体和证书。

:jack_o_lantern: 如果是使用 `BearerTokenAuthenticationFilter` 时，默认是从请求中获取 Authorization header 值。我在实现时使用 cookie 方式，只需要实现 `BearerTokenResolver` 接口。

:jack_o_lantern: 如果是使用 `AbstractAuthenticationProcessingFilter` 时，就自己在 `attemptAuthentication()` 方法中实现。

#### PART3. 组装授权令牌

如果是使用 CAS 或 UsernamePassword 验证，可以跳过这里，因为组装令牌的事情，Filter 里已经实现。就拿 CAS 来比如

```java
// CASAuthenticationFilter
public Authentication attemptAuthentication(HttpServletRequest request, 
                                            HttpServletResponse response)
			throws AuthenticationException, IOException {
    //...
		boolean serviceTicketRequest = serviceTicketRequest(request, response);
		String username = serviceTicketRequest ? CAS_STATEFUL_IDENTIFIER : CAS_STATELESS_IDENTIFIER;
		String password = obtainArtifact(request);
		//...
		UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(
      username, password);
		//...
	}
```

:high_brightness: <font color='yellow'>**在令牌未被验证之前，令牌的初始化必须指定令牌并未完成验证。即 authenticated 属性为 false 。**</font>

:jack_o_lantern: 如果是其他验证方式，则需要自己组装校验令牌，继承 `AbstractAuthenticationToken` 类。

#### PART4. 验证及验证后用户权限信息组装

如果是使用 CAS 验证，需要选择一下 `AbstractCasProtocolUrlBasedTicketValidator` 验证器。当然你也可以实现此 abstract 类，完成 ticket 验证。验证器的调用方是 `CasAuthenticationProvider`。为什么在此介绍 `CasAuthenticationProvider` ？:jack_o_lantern: 因为基本上所有的验证都一定是实现 `AuthenticationProvider` 接口。:jack_o_lantern: 用户权限信息的组装，是实现 `AuthenticationUserDetailsService<T extends Authentication>` 接口，从缓存或是数据库中查询用户或权限点信息组装 UserDetails。

:high_brightness: <font color='yellow'>**验证通过后，可以通过在 Controller 方法上使用 `@AuthenticationPrincipal` 注解，或请求线程里 `SecurityContextHolder.getContext().getAuthentication()`来获取用户授权信息。`@AuthenticationPrincipal` 注解对应 UserDetails 对象，`@CurrentSecurityContext` 注解对应 SecurityContext 对象。**</font>

#### PART5. 验证成功或失败的处理实现

验证成功或失败后的处理方式，一般有几种，重定向首页或登录页面，注册或撤销 JWT token，放行后面的 filter 或 Controller。:jack_o_lantern: 而 JWT token 签发注销的功能，只需实现 `AuthenticationSuccessHandler` 和 `AuthenticationFailureHandler` 两个接口。

#### PART6. 以上内容配置组装

:jack_o_lantern: 配置组装通过继承 `WebSecurityConfigurerAdapter` 类，覆盖 `init(WebSecurity builder)` 方法完成 Filter、provider、handler 等注入。此处不多说，看代码应该就懂了。

以下以 CAS + OAuth2 组合方式的完整代码片段

```java
public class TuscCasTicketValidator extends AbstractCasProtocolUrlBasedTicketValidator {
	 public TuscCasTicketValidator(final String casServerUrlPrefix) {
        // 设定 CAS server
        super(casServerUrlPrefix);
    }
    protected String getUrlSuffix() {
      // 验证路径
      return "validate";
    }
    protected Assertion parseResponseFromServer(final String response)
      throws TicketValidationException {
        // 判定验证通过成果及结果反馈
    }
}
```

```java
// bearer 验证需要
public class CookieBearerTokenResolver implements BearerTokenResolver {
    private static final String COOKIE_NAME_BEARER = "bearer";
    @Override
    public String resolve(HttpServletRequest request) {
        if (request.getCookies() == null) {
            return null;
        }
        return Arrays.stream(request.getCookies())
                .filter(cookie -> StringUtils.equalsAnyIgnoreCase(cookie.getName(), COOKIE_NAME_BEARER))
                .map(Cookie::getValue).findFirst().orElse(null);
    }
}
```

```java
public class MakeTokenHandler extends FilterAuthSuccessHandler {
  public void onAuthenticationSuccess(
    HttpServletRequest request,
    HttpServletResponse response,
    Authentication authentication) throws ServletException, IOException {
    //...
    Jwt jwt = jwtEncoder.encode(
      JwtEncoderParameters.from(jwsHeader, jwtClaimsSetBuilder.build()));
  	String token = jwt.getTokenValue();
		Cookie cookie = new Cookie("bearer", token);
		cookie.setPath("/");
		cookie.setMaxAge(cookieTimeoutSecond);
		cookie.setHttpOnly(false);
		response.addCookie(cookie);
  }
}
public class DefaultAuthFailHandler implements AuthenticationFailureHandler, LogoutSuccessHandler {
	public void onAuthenticationFailure(
    HttpServletRequest request,
    HttpServletResponse response,
    AuthenticationException exception) throws IOException, ServletException {
    log.error("", exception);
    if (exception instanceof InvalidBearerTokenException) {
        Cookie cookie = new Cookie("bearer", null);
        cookie.setMaxAge(0);
        response.addCookie(cookie);
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType("text/plain;charset=utf-8");
        try (PrintWriter writer = response.getWriter()) {
            writer.write("登录已过期或被推出，需要重新登录验证！");
        }
        return;
    }
    // 记住登出或访问前的地址
    response.sendRedirect("登录页面地址");
  }
}
```

```java
// 重新签发 token
public class JwtRenewFilter extends OncePerRequestFilter {
	protected void doFilterInternal(HttpServletRequest request,
                                  HttpServletResponse response,
                                  FilterChain filterChain) 
    throws ServletException, IOException {
    //...
    String bearerToken = bearerTokenResolver.resolve(request);
    Jwt jwt = jwtDecoder.decode(bearerToken);
        Instant expiresAt = jwt.getExpiresAt();
        if (expiresAt.isBefore(Instant.now().plusSeconds(60))) {
            String ticket = jwt.getClaimAsString("st");
            try {
                casTicketValidator.validate(ticket, serviceUrl);
                renewJwt(jwt, response, jwtEncoder);
            } catch (TicketValidationException e) {
              //...
            }
        }
  }
}
```

```java
@Configuration
@EnableWebSecurity
@AutoConfigureAfter(LoginBaseConfiguration.class)
public class LoginConfiguration extends WebSecurityConfigurerAdapter {
    @Resource
    private HttpSecurity httpSecurity;
    @Resource
    private AuthenticationManager authenticationManager;
    @Resource
    private MakeTokenHandler makeTokenHandler
    @Resource
    private DefaultAuthFailHandler defaultAuthFailHandler;
    @Resource
    private AuthenticationDetailsSource<HttpServletRequest, WebAuthenticationDetails> webAuthenticationDetailsSource;
    @Value("${xxxx}")
    private String applicationServerUrl;
    @Value("${xxxx}")
    private String CASServerUrl;
    // 12h
    @Value("${login.jwt.cookieTimeoutSecond:43200}")
    private int cookieTimeoutSecond;
    @Value("${login.jwt.casDurationSecond:600}")
    private int casDurationSecond;
    @Override
    public void init(WebSecurity builder) throws Exception {
        BearerTokenAuthenticationFilter bearerTokenFilter = 
          new BearerTokenAuthenticationFilter(authenticationManager);
      bearerTokenFilter.setAuthenticationDetailsSource(
        dzdaWebAuthenticationDetailsSource);
        BearerTokenResolver bearerTokenResolver = new CookieBearerTokenResolver();
        bearerTokenFilter.setBearerTokenResolver(makeTokenHandler);
        bearerTokenFilter.setAuthenticationFailureHandler(defaultAuthFailHandler);
        AbstractAuthenticationProcessingFilter casAuthenticationFilter
          = new CasAuthenticationFilter();
        casAuthenticationFilter.setAuthenticationFailureHandler(defaultAuthFailHandler);
        casAuthenticationFilter.setAuthenticationManager(authenticationManager);
        casAuthenticationFilter.setFilterProcessesUrl("/login/cas");
      casAuthenticationFilter.setAuthenticationDetailsSource(
        dzdaWebAuthenticationDetailsSource);
      casAuthenticationFilter.setAuthenticationSuccessHandler(
        dzdaCasAuthenticationSuccessHandler);
        JwtRenewFilter jwtRenewFilter = new JwtRenewFilter().
          setBearerTokenResolver(bearerTokenResolver)
          .setCasTicketValidator(new TuscCasTicketValidator(casServerUrl))
          .setServiceUrl(applicationServerUrl)
          .setCookieTimeoutSecond(cookieTimeoutSecond)
          .setCasDurationSecond(casDurationSecond)
          .setLocalDurationSecond(localDurationSecond);
        httpSecurity.authenticationManager(authenticationManager)
                .addFilterBefore(casAuthenticationFilter, 
                                 X509AuthenticationFilter.class)
                .addFilterBefore(jwtRenewFilter, BearerTokenAuthenticationFilter.class)
                .addFilterBefore(bearerTokenFilter, X509AuthenticationFilter.class)
                .authorizeHttpRequests().anyRequest().authenticated()
                .and().anonymous().disable()
                .logout()
                .deleteCookies("bearer", "JESSIONID")
                .invalidateHttpSession(true)
                .logoutUrl("logoutroute")
                .logoutSuccessHandler(new DefaultAuthFailHandler(
                  applicationServerUrl, casServerUrl))
                .and().csrf().disable().cors();
        builder.addSecurityFilterChainBuilder(httpSecurity);
    }
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(Arrays.asList("*"));
        configuration.setAllowedMethods(Arrays.asList("*"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
 		@Bean
    public AuthenticationProvider casAuthenticationProvider(LoginServiceImpl loginService) {
        CasAuthenticationProvider casAuthenticationProvider = new CasAuthenticationProvider();
        casAuthenticationProvider.setAuthenticationUserDetailsService(new XXXCasUserDetailsService(loginService));
        casAuthenticationProvider.setTicketValidator(new TuscCasTicketValidator(ssoUrl));
        ServiceProperties serviceProperties = new ServiceProperties();
        serviceProperties.setService(applicationServerUrl);
        casAuthenticationProvider.setServiceProperties(serviceProperties);
        casAuthenticationProvider.setKey("casAuthenticationProvider");
        return casAuthenticationProvider;
    }
}
```

:high_brightness: <font color='yellow'>**需要明确 Filter 的先后顺序，顺序不对可能会造成验证 bug，比如 renewFilter 应该在 BearerTokenAuthenticationFilter 之前。否则 renewFilter 就没有意义了。**</font>

