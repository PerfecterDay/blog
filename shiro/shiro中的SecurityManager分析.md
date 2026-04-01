
# shiro中的SecurityManager分析
{docsify-updated}




# shiro中的`SecurityManager`分析

## `SecurityManager`接口定义
SecurityManager是shiro中的核心管理类，是一个接口，首先看一下SecurityManager的定义：

    public interface SecurityManager extends Authenticator, Authorizer, SessionManager {
        Subject login(Subject var1, AuthenticationToken var2) throws AuthenticationException;
        void logout(Subject var1);
        Subject createSubject(SubjectContext var1);
    }
`Authenticator`、`Authorizer`、`SessionManager`都是shiro中另外几个组件。`Authenticator`是认证器，负责认证功能；`Authorizer`是鉴权器，负责权限鉴定功能；`SessionManager`负责session管理功能。后续分析各个模块的功能。


## `CachingSecurityManager`实现了 `SecurityManager`

    public abstract class CachingSecurityManager implements SecurityManager, Destroyable, CacheManagerAware, EventBusAware{
        private CacheManager cacheManager;//缓存管理器
        private EventBus eventBus;
        ....
    }
该类增加了缓存管理和事件总线功能。

## `RealmSecurityManager`继承自 `CachingSecurityManager`
该类增加了一个`Collection<Realm>`类型的域，可以保存多个`Realm`对象。

    public abstract class RealmSecurityManager extends CachingSecurityManager {
        private Collection<Realm> realms;
        .....
    }


## `AuthenticatingSecurityManager`继承自 `RealmSecurityManager`
该类增加了`Authenticator`域，使用该认证器可以实现认证功能。

    public abstract class AuthenticatingSecurityManager extends RealmSecurityManager {
        private Authenticator authenticator = new ModularRealmAuthenticator();
        ....
    }
并且，该类真正实现了Authenticator的AuthenticationInfo方法：

    public AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {
        return this.authenticator.authenticate(token);
    }



## `AuthorizingSecurityManager`继承自 `AuthenticatingSecurityManager`

与`AuthenticatingSecurityManager`该类增加了`Authorizer`域，使用该鉴权器可以实现权限验证功能。

    public abstract class AuthorizingSecurityManager extends AuthenticatingSecurityManager {
        private Authorizer authorizer = new ModularRealmAuthorizer();
        ...
    }
调用域对象`authorizer`的相应方法实现了`Authorizer`的所有接口方法。

## `SessionsSecurityManager`继承自`AuthorizingSecurityManager`

    public abstract class SessionsSecurityManager extends AuthorizingSecurityManager {
        private SessionManager sessionManager = new DefaultSessionManager();
        ....
    }
跟前两个类一样，通过增加一个`SessionManager`的域来实现`SessionManager`接口的功能。


## `DefaultSecurityManager`继承自`SessionsSecurityManager`

    public class DefaultSecurityManager extends SessionsSecurityManager {
        protected RememberMeManager rememberMeManager;
        protected SubjectDAO subjectDAO;
        protected SubjectFactory subjectFactory;
        ....
    }
`subjectFactory`对象用来生成`subject`，`subjectDAO`用来保存`subject`,`rememberMeManager`实现rememberMe功能。

当我们在程序中调用`subject.login(token)`方法，会调用`SecurityManager`的`login`方法：

    public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
       AuthenticationInfo info;
       try {
           info = this.authenticate(token);
       } catch (AuthenticationException var7) {
          ........
       }
       Subject loggedIn = this.createSubject(token, info, subject);
       this.onSuccessfulLogin(token, info, loggedIn);
       return loggedIn;
   }
可见会调用`authenticate`方法，上文分析到，`authenticate`方法实际上会调用保存的`Authenticator`的`authenticate`方法，分析`Authenticator`会得知，其最终会调用`Realm`的`getAuthenticationInfo`方法。由此，将登陆操作和`Realm`对象关联起来了.并且，认证成功后，会利用认证信息和token及现在subject生成一个新的`Subject`的已认证对象：
  
       Subject loggedIn = this.createSubject(token, info, subject);
  