
# shiro中的Subject分析
{docsify-updated}




# shiro中的Subject分析
shiro中的Subject对象代表主体，代表了当前“用户”，这个用户不一定是一个具体的人，与当前应用交互的任何东西都是Subject，如网络爬虫，机器人等；即一个抽象概念；所有 Subject 都绑定到 SecurityManager，与 Subject 的所有交互都会委托给 SecurityManager；可以把 Subject 认为是一个门面；SecurityManager 才是实际的执行者。

也就是说对于我们而言，最简单的一个 Shiro 应用：
1、 应用代码通过 Subject 来进行认证和授权，而 Subject 又委托给 SecurityManager；
2、 我们需要给 Shiro 的 SecurityManager 注入 Realm，从而让 SecurityManager 能得到合法的用户及其权限进行判断。


## `Subject`接口

    public interface Subject {
        Object getPrincipal();
        PrincipalCollection getPrincipals();
        boolean isPermitted(String var1);
        boolean hasRole(String var1);
        boolean[] hasRoles(List<String> var1);
        boolean hasAllRoles(Collection<String> var1);
        void checkRole(String var1) throws AuthorizationException;
        void login(AuthenticationToken var1) throws AuthenticationException;
        boolean isAuthenticated();
        boolean isRemembered();
        Session getSession();
        void logout();
        <V> V execute(Callable<V> var1) throws ExecutionException;
        void execute(Runnable var1);
        <V> Callable<V> associateWith(Callable<V> var1);
        Runnable associateWith(Runnable var1);
        void runAs(PrincipalCollection var1) throws NullPointerException, IllegalStateException;
        boolean isRunAs();
        PrincipalCollection getPreviousPrincipals();
        PrincipalCollection releaseRunAs();
        public static class Builder {
            private final SubjectContext subjectContext;
            private final SecurityManager securityManager;
            public Builder() {
                this(SecurityUtils.getSecurityManager());
            }
            public Builder(SecurityManager securityManager) {
                if (securityManager == null) {
                } else {
                    this.securityManager = securityManager;
                    this.subjectContext = this.newSubjectContextInstance();
                    if (this.subjectContext == null) {
                        throw new IllegalStateException("Subject instance returned from 'newSubjectContextInstance' cannot be null.");
                    } else {
                        this.subjectContext.setSecurityManager(securityManager);
                    }
                }
            }
            protected SubjectContext newSubjectContextInstance() {
                return new DefaultSubjectContext();
            }
            protected SubjectContext getSubjectContext() {
                return this.subjectContext;
            }
            public Subject.Builder sessionId(Serializable sessionId) {
                if (sessionId != null) {
                    this.subjectContext.setSessionId(sessionId);
                }

                return this;
            }
            public Subject.Builder host(String host) {
                if (StringUtils.hasText(host)) {
                    this.subjectContext.setHost(host);
                }

                return this;
            }

            public Subject.Builder session(Session session) {
                if (session != null) {
                    this.subjectContext.setSession(session);
                }

                return this;
            }

            public Subject.Builder principals(PrincipalCollection principals) {
                if (!CollectionUtils.isEmpty(principals)) {
                    this.subjectContext.setPrincipals(principals);
                }

                return this;
            }

            public Subject.Builder sessionCreationEnabled(boolean enabled) {
                this.subjectContext.setSessionCreationEnabled(enabled);
                return this;
            }

            public Subject.Builder authenticated(boolean authenticated) {
                this.subjectContext.setAuthenticated(authenticated);
                return this;
            }

            public Subject.Builder contextAttribute(String attributeKey, Object attributeValue) {
                if (attributeKey == null) {
                    String msg = "Subject context map key cannot be null.";
                    throw new IllegalArgumentException(msg);
                } else {
                    if (attributeValue == null) {
                        this.subjectContext.remove(attributeKey);
                    } else {
                        this.subjectContext.put(attributeKey, attributeValue);
                    }

                    return this;
                }
            }

            public Subject buildSubject() {
                return this.securityManager.createSubject(this.subjectContext);
            }
        }
    }


## `DelegatingSubject`

    public class DelegatingSubject implements Subject {
        private static final String RUN_AS_PRINCIPALS_SESSION_KEY =
                DelegatingSubject.class.getName() + ".RUN_AS_PRINCIPALS_SESSION_KEY";
        protected PrincipalCollection principals;
        protected boolean authenticated; //代表是否是认证过的
        protected String host;
        protected Session session; //session对象
        /**
         * @since 1.2
         */
        protected boolean sessionCreationEnabled;
        protected transient SecurityManager securityManager;//委托对象
        .....
    }
`Subject`对象上的操作大部分都委托给`SecurityManager`域对象进行，包括认证、授权等：
授权：

    public boolean isPermitted(String permission) {
        return hasPrincipals() && securityManager.isPermitted(getPrincipals(), permission);
    }

登录认证：

    public void login(AuthenticationToken token) throws AuthenticationException {
        clearRunAsIdentitiesInternal();
        Subject subject = securityManager.login(this, token);
        PrincipalCollection principals;
        String host = null;
        .....
    }
