# Servlet 中的异步请求
{docsify-updated}

> https://www.hackerearth.com/practice/notes/asynchronous-servlets-in-java/  
> https://www.cnblogs.com/okong/p/springboot-twenty.html

Java 应用程序中同时运行过多的线程可能会消耗大量资源。基于网络的 Java 应用程序也不例外。传入的请求由专门的 HTTP 工作线程处理，这些工作线程将处理这些传入的请求，直到最终生成响应并发送回客户端。

在网络应用程序中，预计同时在线的用户数量会非常多，因此资源消耗问题可能会成为活动线程的真正问题。HTTP 工作线程属于一个专用池，可能会被耗尽。此外，线程本身也需要消耗自己的资源，因此将 HTTP 线程池的规模增大到（非常）大也会导致系统资源耗尽：它根本无法扩展。

这时，异步 Servlet 可能会有所帮助。简而言之，异步 Servlet 使应用程序能以异步方式处理传入请求：**一个指定的 HTTP 工作线程处理传入请求，然后将请求传递给另一个后台线程，后者负责处理请求并将响应发送回客户端。初始 HTTP 工作线程将请求传递给后台线程后，会立即返回 HTTP 线程池，以便处理另一个请求。** 

与普通的 Servlet + 自定义线程池方案相比：
+ 异步线程可以直接将请求分派给其它非 HTTP 处理线程处理，自己立马返回到 HTTP 线程池处理其它 HTTP 请求。
+ 普通 Servlet 在一个请求处理完成之前，HTTP 工作线程要一直运行，它只能委托其它线程处理部分任务，等其他线程处理完毕后，拿到异步线程的处理结果返回给客户端（如果需要的话），线程本身必须一直服务到请求处理完毕。
+ 异步请求（AsyncServlet） 最大的价值不是“省线程”，而是 把长时间阻塞 I/O 从容器线程池中（处理客户请求的线程）释放出来，从而显著提升吞吐量与响应能力。

这种方法本身可以**解决 HTTP 线程池耗尽的问题，但无法解决系统资源消耗的问题。毕竟，为处理请求创建了另一个后台线程，因此同时活动的线程数量不会减少，系统资源消耗也不会改善。**

## 同步请求

```java
public class TestSyncServlet extends HttpServlet {

  private static final long serialVersionUID = 1L;

  @Override
  protected void doGet(HttpServletRequest request, 
      HttpServletResponse response) throws ServletException, IOException {

    final long startTime = System.nanoTime();
    try {
      Thread.sleep(2000);
    } catch (InterruptedException e) {
      throw new RuntimeException(e);
    }
    PrintWriter out = response.getWriter();
    out.print("Work completed. Time elapsed: " + (System.nanoTime() - startTime));
    out.flush();

  }

}
```

## 异步请求
```java
public class TestAsyncServlet extends HttpServlet {

  private static final long serialVersionUID = 1L;

  @Override
  protected void doGet(HttpServletRequest request, 
      HttpServletResponse response) throws ServletException, IOException {

    final long startTime = System.nanoTime();
    final AsyncContext asyncContext = request.startAsync(request, response);

    new Thread() {

      @Override
      public void run() {
        try {
          ServletResponse response = asyncContext.getResponse();
          response.setContentType("text/plain");
          PrintWriter out = response.getWriter();
          Thread.sleep(2000);
          out.print("Work completed. Time elapsed: " + (System.nanoTime() - startTime));
          out.flush();
          asyncContext.complete();
        } catch (IOException | InterruptedException e) {
          throw new RuntimeException(e);
        }
      }
    }.start();
  }
}
```

### 核心 AsyncContext 接口