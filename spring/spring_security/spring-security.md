# Spring security
{docsify-updated}

> https://docs.spring.io/spring-security/reference/servlet/architecture.html

## spring-security 整体架构

<center>
<img src="pics/spring-security.png" width="60%">
</center>
上图中  Bean Filter 实际上是 FilterChainProxy 。

Spring security 的配置由两个关键步骤组成：注册过滤器和创建安全规则。  
Spring security 提供了 `AbstractSecurityWebApplicationInitializer` 类来注册过滤器，我们只要继承该类即可实现过滤器的注册。  
使用 springboot 时，只要添加 `spring-boot-starter-security` 依赖，就会自动配置：
1. 一个名为 `springSecurityFilterChain` 的 Filter 类型的 bean (`DelegatingFilterProxyRegistrationBean`)，这个 bean 负责所有安全相关的功能
2. `TokenAuthenticationFilter`
3. 生成一个名为 `UserDetailsService` 的 bean ,且会配置一个名为 user ，密码随机（会打印在控制台）的用户
4. 会把 `springSecurityFilterChain` 的 Filter 注册到 servlet 容器中以实现拦截功能

注册过滤器后，需要对 Spring security 进行安全规则的配置：
```
@EnableWebSecurity
public class WebSecurityConfigure extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/admin/api/**").hasRole("admin")
                .antMatchers("/user/api/**").hasRole("user")
                .antMatchers("/app/api/**").permitAll()
                .anyRequest()
                .authenticated()
                .and()
                .formLogin(form -> form
                .loginPage("/login")  //自定义认证表单
                .permitAll());
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService()).passwordEncoder(new BCryptPasswordEncoder());
    }

    public UserDetailsService userDetailsService(){
        InMemoryUserDetailsManager inMemoryUserDetailsManager = new InMemoryUserDetailsManager();
        inMemoryUserDetailsManager.createUser(User.withUsername("user1").password(new BCryptPasswordEncoder().encode("test")).roles("suer").build());
        inMemoryUserDetailsManager.createUser(User.withUsername("admin").password(new BCryptPasswordEncoder().encode("admin")).roles("admin").build());
        return inMemoryUserDetailsManager;
    }
}
```
`@EnableWebSecurity/@EnableWebMvcSecurity` 注解分别用来启用 Spring security web 和 spring mvc 的集成。使用这两个注解的类必须实现 `WebSecurityConfigurer`接口或者继承 `WebSecurityConfigurerAdapter` 类。

### 认证
1. `SecurityContextHolder`   
   Spring security 用来存储认证过的用户的详细信息的地方。
   <img src="pics/securitycontextholder.png" alt="">  
   Spring security 不关心 `SecurityContextHolder` 是如何生成的，如果其中有值，就会被当成当前认证过的用户。所以，我们可以直接对其进行赋值来生成一个认证过的用户。
2. `SecurityContext`    
   从 `SecurityContextHolder` 获得的保存了当前认证过的用户的 `Authentication` 
3. `Authentication`  
   可以看做是 `AuthenticationManager` 的输入用来提供认证用的凭据信息，或者是当前 SecurityContext 中的认证过的用户。 `Authentication` 包含以下信息：
    1. `principal` : 用户标识（用户名/ID...）
    2. `credentials`: 通常是密码
    3. `authorities`: 授予的权限，比如角色/scope...
4. `GrantedAuthority`    
   代表`Authentication` 的授权信息,标识用户有哪些权限。可以使用 `Authentication.getAuthorities()`获取到。
5. `AuthenticationManager`  
   定义了 Spring Security 的 Filters 如何来执行认证逻辑的 API，它会返回一个 `Authentication`对象，这个对象会设置到 `SecurityContextHolder` 中。
6. `ProviderManager`   
   `AuthenticationManager` 的最常见实现，里边有一个 `List<AuthenticationProvider>` 列表。它会遍历这个列表中的每个 `AuthenticationProvider` 来执行认证逻辑。直到找到一个能够执行认证的 `AuthenticationProvider`。
   <img src="pics/provider_manager.png" alt="">
7. `AuthenticationProvider`  
   被 `ProviderManager` 用来执行特定类型的认证的，比如 `DaoAuthenticationProvider` 支持基于 username/password 的认证，而 `JwtAuthenticationProvider` 支持 JWT token 的认证.
8.  Request Credentials with AuthenticationEntryPoint   
   请求客户端提供认证凭据的认证入口点，通常是返回一个 HTTP 相应给客户端，要求客户端提供认证凭据。
9.  `AbstractAuthenticationProcessingFilter`   
    一个用来做认证的 Filter 的基类。
    

    <center>'
        <img src="pics/AbstractAuthenticationProcessingFilter.png" width="60%">  
    </center> 
    
    1. 当用户提交认证凭据后， `AbstractAuthenticationProcessingFilter` 会根据 `HttpRequest` 创建一个用来做认证的 `Authentication` 对象，具体的 `Authentication` 取决于 `AbstractAuthenticationProcessingFilter`的子类，比如 `UsernamePasswordAuthenticationFilter` 会创建一个 `UsernamePasswordAuthenticationToken`.
    2. `Authentication`会被传递给 `AuthenticationManager` 来做认证
    3. 如果认证失败
       + 会清空 `SecurityContextHolder` 
       + `RememberMeServices.loginFail` 会被调用，如果配置了 remember me 功能的话
       + `AuthenticationFailureHandler` 会被调用
    4. 如果认证成功
       + `SessionAuthenticationStrategy` 会被通知一个新的登录
       + `Authentication`会被设置到 `SecurityContextHolder`，后边 `SecurityContextPersistenceFilter` 会把 `SecurityContext` 保存到 `HttpSession`中
       + `RememberMeServices.loginSuccess` 会被调用，如果配置了 remember me 功能的话
       + `ApplicationEventPublisher`会发布 `InteractiveAuthenticationSuccessEvent`事件


### 授权
常用授权注解：
+ `@DeclareRoles`: 它不在 Spring Security 环境中使用，它提供类一种列出所有应用程序使用的角色的方式。
+ `@DenyAll`: 将禁止所有访问
+ `@PermitAll`: 与 @DenyAll 相反，表示所有人都可以执行该方法。
+ `@RolesAllowed`: 只有拥有指定角色中的一个，才能执行方法。可以标注在类或者方法上。
+ `@RunAs`: 指示容器使用一个不同的用户运行该方法。只在 Java EE 中支持，Spring Security 不支持。
+ `@PreAuthorize`: 注解中指定的安全表达式将在方法执行前执行。该表达式可以通过在参数名前添加#的方式引用任意的方法参数，并可以调用方法参数的方法和访问参数的属性。如果表达式的结果为真，则会执行方法，否则不执行并抛出 AccessDeniedException 异常。
+ `@PostAuthorize`: 注解中指定的安全表达式将在方法执行后执行，不能访问方法参数，但是可以用变量名returnObject 访问方法的返回值。
+ `@PostFilter`: 可以过滤方法的返回值 ，只有返回值是集合或者数组类型时才能使用。集合中的每个元素都会执行一次表达式，可以使用 filterObject 引用每个元素。表达式结果为假的元素会从集合中移除。
+ `@PretFilter`:  与@PostFilter类似，不过它作用于方法参数而不是返回值。

#### 授权原理
1. 访问决策投票者
   访问决策投票者是决策制定过程中的重要参与者。一个典型的 Spring Security 配置将包含几个投票者，他们都是 `AccessDecisionVoter` 的实现。当投票者发表自己的意见表示是否为访问授权时，它根据投票者得到的特定信息进行投票。如果它未得到足够的信息，无法做出合理的投票时，它将在它的 `vote` 方法中返回 `AccessDecisionVoter.ACCESS_ABSTAIN` ，从而放弃投票。如果它获得了足够的信息并相信应该为访问授权，那么它将返回 `AccessDecisionVoter.GRANTED` ，否则，如果不同意授权则返回 `AccessDecisionVoter.DENIED` 。  
   现在有几个可用的 `AccessDecisionVoter` 实现：
   + `org.springframework.security.acls.AclEntryVoter`：根据 Spring security 的访问控列表进行投票。如果被访问的资源没有使用 `hasPermission` 表达式函数进行保护，那么 AclEntryVoter 将放弃投票。该投票者不是自动配置的，实际上可能永远也不会使用它。
   + `org.springframework.security.access.vote.AuthenticatedVoter`：被用作一种特殊情况下的角色: `IS_AUTHENTICATED_ANONYMOUSLY` 、`IS_AUTHENTICATED_REMEMBERED` 、 `IS_AUTHENTICATED_FULLY`。如果资源并未使用这些角色进行保护，那么 AuthenticatedVoter 将会放弃投票
   + `org.springframework.security.access.annotation.Jsr250Voter`: 将为所有使用了 Common Annotations API(JSR 250)进行保护的方法进行投票。只有启用了对 JSR 250 注解的支持，该投票者才可用。只有方法上标注了 JSR 250注解时，它才会投票。
   + `org.springframework.security.access.prepost.PreInvocationAuthorizationAdviceVoter`: 将根据 `@PreAuthorize` 和`@PretFilter` 注解进行投票。只有启用执行前和执行后注解时才会启用它。对于非方法资源或者方法（或者它的类或接口）上不存在注解，它将会弃权。 `@PostAuthorize` 和 `@PostFilter` 没有投票者，因为投票者只在资源访问前使用， Spring security 将把这些注解当做特殊情况处理。
   + `org.springframework.security.access.vote.RoleHierarchyVoter`：使用角色层次系统做出决策，如果启用了角色层次吸引，就必须手动创建它。如果被保护的资源商不存在角色层次限制，那么它将弃权。
   + `org.springframework.security.access.vote.RoleVoter`：将为使用非表达式 URL 限制、方法切点限制或者 `@Secured` 注解保护的资源投票，其它情况它会弃权。只有一个或多个被列出的“角色”都是用了 ROLE—— 前缀时，它才会投票。
   + `org.springframework.security.web.access.expression.WebExpressionVoter`：将根据表达式保护 URL 资源做出它的决策。对于方法保护决策和不使用表达式保护的特定 URL，它会弃权。
  
2. 访问决策管理器
   正如前面所述，现在有几个可用的内建投票者，并且多个投票者可能同时被启用。多个投票者在特定的访问决策上可能都给出了非弃权的意见，那么如何解决这些差异呢？ Spring security 使用访问决策管理器协调这些投票者投出的选票，并将他们转换成访问请求的最终决策。

   Spring security 有3个标准的 `AccessDecisionManager` 实现。默认情况下，所有投票者都弃权时访问将被拒绝，但没个标准实现都提供一个设置，用于决定在这种场景下是否授权访问。

   1. 由赞成结果决定
      `org.springframework.security.access.vote.AffirmativeBased`是默认的配置，也是最简单的决策管理器。只要有一个非弃权的投票者赞成，该决策管理者就授权访问。即使有其它投票者拒绝访问。
   2. 由协商决定
      `org.springframework.security.access.vote.ConsensusBased`决策管理器基于简单的多数原则，如果51%的非弃权投票者赞成就授予访问权限。
   3. 由全体决定
      `org.springframework.security.access.vote.UnanimousBased`，它要求所有的非弃权投票则都投赞成票，否则将拒绝访问请求。
   
   要配置一个 `AccessDecisionManager`，只需要声明一个标注了 `@Configuration` 的配置类，让该类扩展 `GlobalMethodSecurityConfiguration` 类，并且覆盖 `accessDecisionManager` 方法返回一个 `ConsensusBased`、`UnanimousBased`或者一些自定义的实现。


## 配置
Block_chain_managementSecurityAutoConfiguration 配置类

AuthorizationManagerBeforeMethodInterceptor
      PreAuthorizeAuthorizationManager
        ExpressionUtils
         SecurityFrameworkServiceImpl

TokenAuthenticationFilter 尝试用 token 获取用户信息，获取到就会设置到

SecurityFrameworkUtils.setLoginUser  

SecurityContextHolder.getContext().setAuthentication(authentication);  



Principal - This interface represents the abstract notion of a principal, which can be used to represent any entity, such as an individual, a corporation, and a login id.
public String getName();

Authentication
Represents the token for an authentication request or for an authenticated principal once the request has been processed by the AuthenticationManager.authenticate(Authentication) method.
Once the request has been authenticated, the Authentication will usually be stored in a thread-local SecurityContext managed by the SecurityContextHolder by the authentication mechanism which is being used. An explicit authentication can be achieved, without using one of Spring Security's authentication mechanisms, by creating an Authentication instance and using the code:
  SecurityContext context = SecurityContextHolder.createEmptyContext();
  context.setAuthentication(anAuthentication);
  SecurityContextHolder.setContext(context);
  
Note that unless the Authentication has the authenticated property set to true, it will still be authenticated by any security interceptor (for method or web invocations) which encounters it.
In most cases, the framework transparently takes care of managing the security context and authentication objects for you.


Block_chain_managementWebSecurityConfigurerAdapter 定义了一个 SecurityFilterChain
WebSecurityConfiguration#springSecurityFilterChain ---> FilterChainProxy
webSecurityExpressionHandler
springSecurityFilterChain

## 为什么 rpc-api 的请求不用 token ？
AuthorizationFilter
RequestMatcherDelegatingAuthorizationManager
AuthorizeHttpRequestsConfigurer


```
cn.com.demo.module.system.framework.security.config.SecurityConfiguration
cn.com.demo.module.infra.framework.security.config.SecurityConfiguration
```

