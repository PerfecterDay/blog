
# shiro中的ProxiedFilterChain分析
{docsify-updated}




# shiro中的ProxiedFilterChain分析
shiro中的过滤器链和Servlet中的FilterChain之间的关系是什么呢？就要看`ProxiedFilterChain`代理对象如何封装FilterChain了：

    public class ProxiedFilterChain implements FilterChain {
        private FilterChain orig;
        private List<Filter> filters;
        private int index = 0;


        public ProxiedFilterChain(FilterChain orig, List<Filter> filters) {
            if(orig == null) {
                throw new NullPointerException("original FilterChain cannot be null.");
            } else {
                this.orig = orig;
                this.filters = filters;
                this.index = 0;
            }
        }

        public void doFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {
            if(this.filters != null && this.filters.size() != this.index) {
                if(log.isTraceEnabled()) {
                    log.trace("Invoking wrapped filter at index [" + this.index + "]");
                }

                ((Filter)this.filters.get(this.index++)).doFilter(request, response, this);
            } else {
                if(log.isTraceEnabled()) {
                    log.trace("Invoking original filter chain.");
                }

                this.orig.doFilter(request, response);
            }

        }
    }

    看源码很简单，`ProxiedFilterChain`用一个`FilterChain`类型域保存Servlet的原生过滤器链，用一个List<Filter>列表保存自定义过滤器链。

    重写`doFilter`方法中，首先执行自定义过滤器链，然后执行原生过滤器链.