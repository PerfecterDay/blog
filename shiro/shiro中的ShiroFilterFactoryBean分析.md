
# shiro中的ShiroFilterFactoryBean分析
{docsify-updated}




# shiro中的`ShiroFilterFactoryBean`分析
shiro中的`ShiroFilterFactoryBean`是配置shiro拦截主要对象，从名字可以看出它是一个工厂类，看一下它的定义：
    
    public class ShiroFilterFactoryBean implements FactoryBean, BeanPostProcessor {
        private SecurityManager securityManager;
        private Map<String, Filter> filters = new LinkedHashMap();
        private Map<String, String> filterChainDefinitionMap = new LinkedHashMap();
        private String loginUrl;
        private String successUrl;
        private String unauthorizedUrl;
        private AbstractShiroFilter instance;
    }
`ShiroFilterFactoryBean`中有一个`SecurityManager`域，用两个`Map`保存自定义的`Filter`和拦截器链的配置。

定义一个FactoryBean时，其实真正生成bean的是`getObject`方法，看一下`ShiroFilterFactoryBean`的`getObject`方法：

    public Object getObject() throws Exception {
        if(this.instance == null) {
            this.instance = this.createInstance();
        }
        return this.instance;
    }

    protected AbstractShiroFilter createInstance() throws Exception {
        SecurityManager securityManager = this.getSecurityManager();
        String manager1;
        if(securityManager == null) {
           ....
        } else if(!(securityManager instanceof WebSecurityManager)) {
            ....
        } else {
            FilterChainManager manager = this.createFilterChainManager();
            PathMatchingFilterChainResolver chainResolver = new PathMatchingFilterChainResolver();
            chainResolver.setFilterChainManager(manager);
            return new ShiroFilterFactoryBean.SpringShiroFilter((WebSecurityManager)securityManager, chainResolver);
        }
    }

可见`ShiroFilterFactoryBean`的主要功能是生成一个`SpringShiroFilter`对象Bean.在创建Benn的过程中，分别创建了`FilterChainManager`、`PathMatchingFilterChainResolver`对象。

看一下`FilterChainManager`的创建过程:

    protected FilterChainManager createFilterChainManager() {
        DefaultFilterChainManager manager = new DefaultFilterChainManager();
        Map defaultFilters = manager.getFilters();
        Iterator filters = defaultFilters.values().iterator();

        while(filters.hasNext()) {
            Filter chains = (Filter)filters.next();
            this.applyGlobalPropertiesIfNecessary(chains);
        }

        Map filters1 = this.getFilters();
        String entry1;
        Filter url;
        if(!CollectionUtils.isEmpty(filters1)) {
            for(Iterator chains1 = filters1.entrySet().iterator(); chains1.hasNext(); manager.addFilter(entry1, url, false)) {
                Entry entry = (Entry)chains1.next();
                entry1 = (String)entry.getKey();
                url = (Filter)entry.getValue();
                this.applyGlobalPropertiesIfNecessary(url);
                if(url instanceof Nameable) {
                    ((Nameable)url).setName(entry1);
                }
            }
        }

        Map chains2 = this.getFilterChainDefinitionMap();
        if(!CollectionUtils.isEmpty(chains2)) {
            Iterator entry2 = chains2.entrySet().iterator();

            while(entry2.hasNext()) {
                Entry entry3 = (Entry)entry2.next();
                String url1 = (String)entry3.getKey();
                String chainDefinition = (String)entry3.getValue();
                manager.createChain(url1, chainDefinition);
            }
        }

        return manager;
    }
注意一下这句for循环语句：

    for(Iterator chains1 = filters1.entrySet().iterator(); chains1.hasNext(); manager.addFilter(entry1, url, false){...}
这段代码会把`ShiroFilterFactoryBean`的`filters`域中保存的拦截器添加到生成的`DefaultFilterChainManager`对象的`filters`域。

接着，会解析`filterChainDefinitionMap`域，并根据其中的内容，生成拦截器链：

    manager.createChain(url1, chainDefinition);
