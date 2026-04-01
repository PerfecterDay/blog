
# shiro中的Authenticator认证器
{docsify-updated}




# shiro中的`Authenticator`认证器

## `Authenticator`接口
`Authenticator`接口是shiro中用来实现认证功能的接口：

    public interface Authenticator {
        AuthenticationInfo authenticate(AuthenticationToken var1) throws AuthenticationException;
    }
接口中只有一个`authenticate`方法。该方法接收一个`AuthenticationToken`类型的参数，返回一个`AuthenticationInfo`类型对象。

## `AbstractAuthenticator`抽象类
    
    public abstract class AbstractAuthenticator implements Authenticator, LogoutAware {
        private Collection<AuthenticationListener> listeners = new ArrayList();
        ....
    }
该类添加了`Collection<AuthenticationListener>`类型的listeners域，`AuthenticationListener`定义如下：

    public interface AuthenticationListener {
        void onSuccess(AuthenticationToken var1, AuthenticationInfo var2);

        void onFailure(AuthenticationToken var1, AuthenticationException var2);

        void onLogout(PrincipalCollection var1);
    }
不难看出，该类主要是用来监听认证状态的功能。

该类实现了`authenticate`方法：

    public final AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {
            if (token == null) {
            } else {
                AuthenticationInfo info;
                try {
                    info = this.doAuthenticate(token);
                } catch (Throwable var8) {
                    try {
                        this.notifyFailure(token, ae);
                    } catch (Throwable var7) {
                      ....  
                    }

                    throw ae;
                }
                this.notifySuccess(token, info);
                return info;
            }
        }
    protected abstract AuthenticationInfo doAuthenticate(AuthenticationToken var1) throws AuthenticationException;
该方法调用了`doAuthenticate`方法，`doAuthenticate`是个抽象方法，子类可以实现此方法，自定义认证功能。调用`doAuthenticate`之后，根据调用状态会调用`notifyFailure`和`notifySuccess`方法：

    protected void notifySuccess(AuthenticationToken token, AuthenticationInfo info) {
            Iterator var3 = this.listeners.iterator();

            while(var3.hasNext()) {
                AuthenticationListener listener = (AuthenticationListener)var3.next();
                listener.onSuccess(token, info);
            }

        }

        protected void notifyFailure(AuthenticationToken token, AuthenticationException ae) {
            Iterator var3 = this.listeners.iterator();

            while(var3.hasNext()) {
                AuthenticationListener listener = (AuthenticationListener)var3.next();
                listener.onFailure(token, ae);
            }

        }
这两个方法会调用`listeners`监听器中的相应方法。

## `ModularRealmAuthenticator`继承自`AbstractAuthenticator`
    public class ModularRealmAuthenticator extends AbstractAuthenticator {
        private Collection<Realm> realms;
        private AuthenticationStrategy authenticationStrategy = new AtLeastOneSuccessfulStrategy();
        ....
    }
该类增加了`Collection<Realm>`和`AuthenticationStrategy`类型的两个域，前者将`Authenticator`和`Realm`接口关联起来，后者用于实现自定义的认证策略。

看一下`doAuthenticate`方法：
    
    protected AuthenticationInfo doAuthenticate(AuthenticationToken authenticationToken) throws AuthenticationException {
            this.assertRealmsConfigured();
            Collection<Realm> realms = this.getRealms();
            return realms.size() == 1 ? this.doSingleRealmAuthentication((Realm)realms.iterator().next(), authenticationToken) : this.doMultiRealmAuthentication(realms, authenticationToken);
        }
该方法根据保存的`realms`的数量，去决定调用`doSingleRealmAuthentication`还是`doMultiRealmAuthentication`方法：

    protected AuthenticationInfo doSingleRealmAuthentication(Realm realm, AuthenticationToken token) {
        if (!realm.supports(token)) {
           ... 失败
        } else {
            AuthenticationInfo info = realm.getAuthenticationInfo(token);
            if (info == null) {
               ...失败
            } else {
                return info;
            }
        }
    }
单`realm`认证时，只是简单调用`realm`的`getAuthenticationInfo`方法，并以此决定认证成功还是失败。单`realm`时，`AuthenticationStrategy`并没有用到。只有在多`realm`认证时才会用到`AuthenticationStrategy`:

    protected AuthenticationInfo doMultiRealmAuthentication(Collection<Realm> realms, AuthenticationToken token) {
        AuthenticationStrategy strategy = this.getAuthenticationStrategy();
        AuthenticationInfo aggregate = strategy.beforeAllAttempts(realms, token);
        Iterator var5 = realms.iterator();
        while(var5.hasNext()) {
            Realm realm = (Realm)var5.next();
            aggregate = strategy.beforeAttempt(realm, token, aggregate);
            if (realm.supports(token)) {
                AuthenticationInfo info = null;
                try {
                    info = realm.getAuthenticationInfo(token);
                } catch (Throwable var11) {
                   ...
                    }
                }
                aggregate = strategy.afterAttempt(realm, token, info, aggregate, t);
            } else {
            }
        }
        aggregate = strategy.afterAllAttempts(token, aggregate);
        return aggregate;
    }
使用`AuthenticationStrategy`可以控制在所有`realm`认证开始前、后以及在某个支持`realm`认证开始前后实现一些功能。因此可以控制在某个`realm`失败时该如何处理。

