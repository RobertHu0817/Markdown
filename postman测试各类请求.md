<table>
    <tr>
        <th colspan="2"></th>
        <th>Content-Type</th>
        <th>getParameter(@RequestParam)</th>
        <th>getInputStream</th>
        <th>@RequestBody</th>
    </tr>
    <tr>
        <th rowspan="2">Query Params</th>
        <th>GET</th>
        <th rowspan="2">text/plain</th>
        <th rowspan="2">√</th>
        <th rowspan="2">×</th>
        <th>Required request body is missing</th>
    </tr>
    <tr>
        <th>POST</th>
        <th>Content type 'text/plain;charset=UTF-8' not supported</th>
    </tr>
    <tr>
        <th rowspan="2">form-data</th>
        <th>GET</th>
        <th rowspan="2">multipart/form-data; boundary=--------------------------438387563082110319045207</th>
        <th>×</th>
        <th>----------------------------438387563082110319045207Content-Disposition: form-data; name="a"1----------------------------438387563082110319045207Content-Disposition: form-data; name="b"2----------------------------438387563082110319045207--</th>
        <th>Required request body is missing</th>
    </tr>
    <tr>
        <th>POST</th>
        <th>√</th>
        <th>×</th>
        <th>Content type 'multipart/form-data;boundary=--------------------------869483803786294671434635;charset=UTF-8' not supported</th>
    </tr>
    <tr>
        <th rowspan="2">x-www-form-urlencoded</th>
        <th>GET</th>
        <th rowspan="2">application/x-www-form-urlencoded</th>
        <th>×</th>
        <th>a=1&b=2</th>
        <th>Required request body is missing</th>
    </tr>
    <tr>
        <th>POST</th>
        <th>√</th>
        <th>×</th>
        <th>Content type 'application/x-www-form-urlencoded;charset=UTF-8' not supported</th>
    </tr>
    <tr>
        <th rowspan="2">raw</th>
        <th>GET</th>
        <th rowspan="2">取自header中的Content-Type</th>
        <th rowspan="2">×</th>
        <th rowspan="2">{    "a": 1,    "b": 2}</th>
        <th>content-type=application/json：√<br>
            content-type=其他：Required request body is missing</th>
    </tr>
    <tr>
        <th>POST</th>
        <th>content-type=application/json：√<br>
            content-type=其他：Content type xxx not supported</th>
    </tr>
</table>

### form-data请求格式：boundary=${boundary}用以分隔请求头与请求体，请求体内容各字段之间以--${boundary}来进行分割,以--${boundary}--来结束请求体内容



## Tomcat的Coyote框架

Coyote是Tomcat的Connector框架的名字，简单说就是coyote来处理底层的socket，并将http请求、响应等字节流层面的东西，包装成Request(org.apache.coyote.Request)和Response(org.apache.coyote.Request)两个类供容器使用

* DEBUG查看request：

  ``` java
  -request= SessionRepositoryFilter$SessionRepositoryRequestWrapper
  	-request= RequestFacade
  		-request= Request
  			-coyoteRequest= Request
  				-attributes= HashMap<K,V>
  				-contentTypeMB= MessageBytes
  				-decodedUriMB= MessageBytes (/leap-crm/test/hi)
  				-methodMb= MessageBytes (POST/GET)
  				-parameters= Parameters
  					-paramHashValues= LinkedHashMap<K,V> (request.getParameter正是取此处数据)
  			-parameterMap= ParameterMap<K,V>
  			-parametersParsed= false
  ```

* org.apache.coyote.Request 类注释：

  `Most fields are GC-free, expensive operations are delayed until the  user code needs the information`

  1. 注释告诉我们，Request的大部分字段是“GC free”的，即很少、甚至不会被垃圾回收。杜绝了java中最大的一个性能瓶颈，Request自然性能得到大幅提升

  2. Coyote框架中，接收到request时尽量不做任何处理，使其保持原始的字节状态，直到用户代码需要时才进行需要的部分进行解析转换。例：（前提：使用GET请求，原因见下）在调用`request.getParameter("a")`之前，request中parametersParsed=false，parameters为null；调用时，通过parseParameters()方法解析parameter，解析后request中parametersParsed=true，parameters为a=1 b=2【将一些耗时操作延迟到用户代码一级，挖掘出更多性能潜力，其思想与“延迟加载”如出一辙】

  3. 另一篇文章：

     1）由于Java自动内存回收机制效率不高，有很多问题，所以Coyote中通过recycle机制的使用，及时进行参数初始化和内存释放。以Request为例，Recycle函数中主要进行了三个方面的事情：自身参数的初始化，自身创建的资源的释放和调用类中使用的引用对象的recycle函数。通过递归地进行recycle，一方面及时并且全面地释放了不再需要的资源，另一方面及时对相关参数进行初始化，提高下一次类访问的执行速度。

     2）对于底层的、和字节流打交道的DO（data object），性能瓶颈在于对内存的使用上（因为字节都是放在一块块的内存中），如果能有效的使用内存，就能有效地提高DO的性能。ByteChunk和MessageBytes都是Tomcat为了提高处理效率封装出来的对字节流和字节数组进行优化的类，在执行效率上比java的string和byte数组要高。这两个类都都在org.apache.tomcat.util.buf包中。

     3）由于ByteChunk和MessageBytes的使用，Request中字段的一些耗时操作都会延迟到用户代码一级。也就是说，tomcat内部在使用Request时，都会尽量保证它的字段处于原始的字节状态，直到用户代码（servlet代码层）需要时才进行转换，如果用不到（其实http请求的大部分字段在我们编程时都用不到），就不作转换。这样又进一步挖掘出更多的性能潜力，其思想和“延迟加载”的设计模式如出一辙。

* org.apache.catalina.connector.Request的parseParameters()方法：

  ```java
  protected void parseParameters() {
  
          parametersParsed = true;
  
          Parameters parameters = coyoteRequest.getParameters();
          boolean success = false;
          try {
              
              ...
  
              // charset START
              Charset charset = getCharset();
  
              boolean useBodyEncodingForURI = connector.getUseBodyEncodingForURI();
              parameters.setCharset(charset);
              if (useBodyEncodingForURI) {
                  parameters.setQueryStringCharset(charset);
              }
              // charset END
  
              //解析url参数
              parameters.handleQueryParameters();
  
              ...
  
              // 非POST请求直接返回，不再往下解析->GET请求仅能获取url参数
              if( !getConnector().isParseBodyMethod(getMethod()) ) {
                  success = true;
                  return;
              }
  
              // Content-Type START
              String contentType = getContentType();
              if (contentType == null) {
                  contentType = "";
              }
              int semicolon = contentType.indexOf(';');
              if (semicolon >= 0) {
                  contentType = contentType.substring(0, semicolon).trim();
              } else {
                  contentType = contentType.trim();
              }
              // Content-Type END
  
              // 解析multipart/form-data
              if ("multipart/form-data".equals(contentType)) {
                  parseParts(false);// 解析请求报文，含文件解析处理算法
                  success = true;
                  return;
              }
  
              //Content-Type既不是"multipart/form-data"也不是"application/x-www-form-urlencoded"的直接返回
              if (!("application/x-www-form-urlencoded".equals(contentType))) {
                  success = true;
                  return;
              }
  
              // 解析application/x-www-form-urlencoded
              // 两种请求报文传输方案：1.利用Content-Length；2.Transfer-Encoding:chunked——详见<https://imququ.com/post/transfer-encoding-header-in-http.html>
              int len = getContentLength();
              
              if (len > 0) {
                  // 1.Content-Length
                  ...
                  
                  parameters.processParameters(formData, 0, len);// processParameters方法实际就是使用字符截取解析的方法获得请求报文中parameter的name和value——格式：a=1&b=2
              } else if ("chunked".equalsIgnoreCase(
                      coyoteRequest.getHeader("transfer-encoding"))) {
                  // 2.Transfer-Encoding:chunked
                  ...
                  
                  if (formData != null) {
                      parameters.processParameters(formData, 0, formData.length);
                  }
              }
              success = true;
          } finally {
              if (!success) {
                  parameters.setParseFailedReason(FailReason.UNKNOWN);
              }
          }
  
      }
  ```

  由源码可知：

  1. url参数一定会被解析
  2. 请求中的报文只有在使用POST请求时才会被解析
  3. Content-Type 非 multipart/form-data 和 application/x-www-form-urlencoded（如：application/json）的请求，请求报文会被忽略

* request输入流只能读取一次

  当我们调用`getInputStream()`方法获取输入流时得到的是一个InputStream对象，而实际类型是ServletInputStream，它继承于InputStream。

  InputStream的`read()`方法内部有一个position，标识当前流被读取到的位置，并且随着读取来进行移动，如果读到最后，`read()`会返回-1标识已经读取完了。如果想要重新读取，则需要调用`reset()`方法，position就会移动到上次调用mark的位置，mark默认是0，所以就能从头再读了。调用`reset()`方法的前提是已经重写了`reset()`方法。此外能否reset也是有条件的，它取决于`markSupported()`方法是否返回true。

  而InputStream默认不实现`reset()`，并且`markSupported()`默认也是返回false；而实现类ServletInputStream中也没有重写`mark()`、`reset()`及`markSupported()`方法——因此，从request对象中获取的输入流只能读取一次

  - [ ] > InputStream的`read()`方法内部有一个position????看源码没看到啊啊啊

* HiddenHttpMethodFilter.java

  为应对当时浏览器只支持GET和POST（目前现代浏览器都支持PUT/DELETE，但form表单**似乎**仍只支持GET/POST），Spring3.0添加了一个过滤器 HiddenHttpMethodFilter，可以将PUT/DELETE等POST请求转换为标准的http方法（实际就是发送POST请求的时候加上一个额外的`_method`字段，标注真实的HTTP method）。而在Springboot中，默认使用了此过滤器

  ```java
  /**
   * {@link javax.servlet.Filter} that converts posted method parameters into HTTP methods,
   * retrievable via {@link HttpServletRequest#getMethod()}. Since browsers currently only
   * support GET and POST, a common technique - used by the Prototype library, for instance -
   * is to use a normal POST with an additional hidden form field ({@code _method})
   * to pass the "real" HTTP method along. This filter reads that parameter and changes
   * the {@link HttpServletRequestWrapper#getMethod()} return value accordingly.
   * Only {@code "PUT"}, {@code "DELETE"} and {@code "PATCH"} HTTP methods are allowed.
   ...
   */
  public class HiddenHttpMethodFilter extends OncePerRequestFilter {
  	
      ...
      
      /** Default method parameter: {@code _method} */
  	public static final String DEFAULT_METHOD_PARAM = "_method";
  
  	private String methodParam = DEFAULT_METHOD_PARAM;
      
      ...
      
      @Override
  	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
  			throws ServletException, IOException {
  
  		HttpServletRequest requestToUse = request;
  
           // 由此可见POST方法在进入controller中被使用前就会先被解析一次（request.getParameter(this.methodParam)），导致用户代码中再使用request.getInputStream()时得到的为空流（只能读取一次）
  		if ("POST".equals(request.getMethod()) && request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) == null) {
  			String paramValue = request.getParameter(this.methodParam);
  			if (StringUtils.hasLength(paramValue)) {
  				String method = paramValue.toUpperCase(Locale.ENGLISH);
  				if (ALLOWED_METHODS.contains(method)) {
  					requestToUse = new HttpMethodRequestWrapper(request, method);
  				}
  			}
  		}
  
  		filterChain.doFilter(requestToUse, response);
  	}
  }
  ```

  这就导致了测试表格中POST请求getInputStream()为空的情况。

  - [ ] 为什么RequestBody可以取到POST请求中的数据——HTTP请求处理流程待研究
