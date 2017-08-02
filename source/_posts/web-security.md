title: 记一次web安全漏洞分析与处理
author: libaogang
tags:
  - web
  - 安全
categories:
  - 技术
thumbnail: /img/random/material-18.png
date: 2017-07-29 14:31:00
---
> 记一次用Appsan扫描站点扫描出的漏洞与解决方案

## HTTP PUT方法站点篡改

### 原理分析

通过HTTP PUT 或者DELETE方法，可能会在 Web 服务器上上载、修改或删除 Web 页面、脚本和文件，这通常意味着完全破坏服务器及其内容。

### 解决方案

 - 方案一

修改DefaultServlet初始化参数readonly为true，对web服务器上的文件访问权限会变为只读，因此如PUT 和 DELETE的HTTP命令将被拒绝执行。

```xml
<servlet>
          <servlet-name>default</servlet-name>
          <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
          <init-param>
              <param-name>debug</param-name>
              <param-value>0</param-value>
          </init-param>
          <init-param>
              <param-name>listings</param-name>
              <param-value>false</param-value>
          </init-param>
          <init-param>
              <param-name>readonly</param-name>
              <param-value>true</param-value>
          </init-param>
          <load-on-startup>1</load-on-startup>
</servlet>
```  

- 方案二

在web.xml中加入以下配置，禁止PUT DELETE HEAD OPTIONS TRACE方法
 
```xml
<security-constraint>  
       <web-resource-collection>  
           <url-pattern>/*</url-pattern>  
           <http-method>PUT</http-method>  
           <http-method>DELETE</http-method>  
           <http-method>HEAD</http-method>  
           <http-method>OPTIONS</http-method>  
           <http-method>TRACE</http-method>  
       </web-resource-collection>  
       <auth-constraint></auth-constraint>  
</security-constraint>  
```

## 跨站脚本攻击(Cross Site Scripting)
xss是一门热门又不太受重视的Web攻击手法，因为这是一种被动的攻击方式，耗时长，有一定几率不成功，且没有相应的软件来完成自动化攻击。但是这并不影响黑客对此攻击手法的偏爱，因为几乎每个网站都存在xss漏洞。
### 原理分析
本地搭建一个java web环境，新建xss.jsp文件，代码如下
```html
<!DOCTYPE>
<html> 
<head> 
<title>xss</title> 
</head> 
<body> 
<form action="" method="get"> 
	<input type="text" name="xss"> 
	<input type="submit"> 
</form> 
<hr>
your input is: <%=request.getParameter("xss")%>  
</body> 
</html> 
```  
页面是这样
![](/image/web-security-1.png)
我们输入123，结果是这样的，没有任何问题
![](/image/web-security-2.png)
看源代码，我们输入的字符串被原封不动地输出了
```html
<!DOCTYPE>
<html> 
<head> 
<title>xss</title> 
</head> 
<body> 
<form action="" method="get"> 
	<input type="text" name="xss"> 
	<input type="submit"> 
</form> 
<hr>
your input is: 123  
</body> 
</html> 
```  
如果我们输入这段代码会怎么样？

```javascript
<script>alert("css")</script>
```  

因为页面会原封不动输出我们的输入，预期的页面的源代码应该是这样的

```html
<!DOCTYPE>
<html> 
<head> 
<title>xss</title> 
</head> 
<body> 
<form action="" method="get"> 
	<input type="text" name="xss"> 
	<input type="submit"> 
</form> 
<hr>
your input is: <script>alert("css")</script>  
</body> 
</html> 
```
我们的脚本会被执行，即弹窗，而事实也确实是这样的，成功弹窗。
![](/image/web-security-3.png)

这时候已经可以确定存在xss漏洞了。其实到这里我也有点疑惑，弹个窗能有多大危害？其实弹窗只是测试，只要确定存在xss漏洞，那么就可以利用这个漏洞来插入并执行我们准备好的脚本，获取cookie从而获取用户权限，操纵用户行为。

当然，这只是基本原理。xss攻击方式多种多样，这里不作一一探索。

### 解决方案

新建XssHttpServletRequestWrapper类，继承自HttpServletRequestWrapper，并覆盖以下方法，将危险字符转义或去除。

```java
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletRequestWrapper;  

public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {  
    
    public XssHttpServletRequestWrapper(HttpServletRequest servletRequest) {
        super(servletRequest);
    }
    
    @Override
    public String[] getParameterValues(String parameter) {
      String[] values = super.getParameterValues(parameter);
      if (values==null){
          return null;
      }
      int count = values.length;
      String[] encodedValues = new String[count];
      for (int i = 0; i < count; i++) {
          encodedValues[i] = cleanXSS(values[i]);
       }
      return encodedValues;
    }
    
    @Override
    public String getParameter(String parameter) {
          String value = super.getParameter(parameter);
          if (value == null) {
                 return null;
                  }
          return cleanXSS(value);
    }
    
    @Override
    public String getHeader(String name) {
        String value = super.getHeader(name);
        if (value == null)
            return null;
        return cleanXSS(value);
    }
    private String cleanXSS(String value) {
        //You'll need to remove the spaces from the html entities below
        value = value.replaceAll("<", "<").replaceAll(">", ">");
        value = value.replaceAll("\\(", "(").replaceAll("\\)", ")");
        value = value.replaceAll("'", "'");
        value = value.replaceAll("eval\\((.*)\\)", "");
        value = value.replaceAll("[\\\"\\\'][\\s]*javascript:(.*)[\\\"\\\']", "\"\"");
        value = value.replaceAll("script", "");
        return value;
    }
    
}
```  
新建XSSFilter类

```java
import java.io.IOException;
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;

public class XSSFilter implements Filter {  
    @Override 
    public void init(FilterConfig filterConfig) throws ServletException {  

    }  

    @Override 
    public void doFilter(ServletRequest request, ServletResponse response,  
            FilterChain chain) throws IOException, ServletException {
        
        System.out.println("enter xssfilter----------------");
        XssHttpServletRequestWrapper xssRequest = new XssHttpServletRequestWrapper(  
                (HttpServletRequest) request); 
        
        chain.doFilter(xssRequest, response);  
    }  

    @Override 
    public void destroy() {  

    }  

}  
```  
web.xml配置如下，拦截所有请求。

```xml
<filter>
     <filter-name>XSSFilter</filter-name>
     <filter-class>xxx.xxx.XSSFilter</filter-class>
</filter>

 <filter-mapping>
     <filter-name>XSSFilter</filter-name>
     <url-pattern>/*</url-pattern>
 </filter-mapping>
```

## 跨站请求伪造 (Cross Site Request Forgery)

如果说xss的本质是攻击者获取了用户身份进行恶意操作，而csrf的本质是借用用户身份进行而已恶意操作。

### 原理分析
看下面一个例子：
A在银行有一笔存款，可以通过请求 http://www.bank.com/transfer?account=A&amount=1000000&for=B 把钱转到B的账户下。C在该银行也有账户，于是他伪造了一个地址 http://www.bank.com/transfer?account=A&amount=1000000&for=C 。如果直接访问，服务器会根据客户端cookie和服务端session识别出当前登录用户是C而不是A，不能接受请求。于是C新建一个广告页面，将此链接伪造在广告下，诱使A自己点这个链接。如果A在这个广告页点了这个链接，那么就会在他不知情的情况下将存款转给了C。

为什么会这样？因为受害者首先已经登录了银行网站取得了合法身份。在浏览器进程的生命周期内，即使浏览器同一个窗口打开了新的tab页面，cookie也都是有效的，在浏览器同一个窗口的多个tab页面里面是共享的。所以当A在广告页点击了C伪造的请求时，同时也携带了A在银行tab页的合法身份。而此时恰好服务端session还没有过期，于是服务器成功响应请求。

可以看到，以上例子是一个get请求，其实无论是get、post请求，链接、表单、ajax请求都无法避免跨站请求伪造的问题。

### 解决方案
- 方案一 通过referer 判定来源页面

```java
String referer=httpRequest.getHeader("referer");
if (referer!= null  && !referer.startsWith("http://www.bank.com") {
	return;
}
```  
从以上例子可以看出，A在点击伪造的请求时肯定是不在http://www.bank.com 页面的，因此可以通过判断来源页面来拦截伪造的请求。但是referer是存在于http请求头部的， 可以拦截请求并任意修改referer值。虽然这个例子中受害者不可能自己去修改referer，但终究还是可能存在安全隐患。

- 方案二 令牌同步

Synchronizer token pattern

令牌同步模式（Synchronizer token pattern，简称STP）是在用户请求的页面中的所有表单中嵌入一个token，在服务端验证这个token的技术。token可以是任意的内容，但是一定要保证无法被攻击者猜测到或者查询到。攻击者在请求中无法使用正确的token，因此可以判断出未授权的请求。

Cookie-to-Header Token

对于使用Js作为主要交互技术的网站，将CSRF的token写入到cookie中 

```javascript
Set-Cookie: CSRF-token=i8XNjC4b8KVok4uw5RftR38Wgp2BFwql; expires=Thu, 23-Jul-2015 10:25:33 GMT; Max-Age=31449600; Path=/
```  
然后使用javascript读取token的值，在发送http请求的时候将其作为请求的header

```javascript
X-CSRF-Token: i8XNjC4b8KVok4uw5RftR38Wgp2BFwql
``` 
最后服务器验证请求头中的token是否合法。
