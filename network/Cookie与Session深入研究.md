文章来源：https://github.com/Zeb-D/my-review ，请star强力支持。

# Cookie和Session详解

会话（Session）跟踪是Web程序中常用的技术，用来**跟踪用户的整个会话**。常用的会话跟踪技术是Cookie与Session。**Cookie通过在客户端记录信息确定用户身份**，**Session通过在服务器端记录信息确定用户身份**。

我们接下来从cookie、session机制来分析什么时候使用，怎么使用。

## Cookie机制

在程序中，会话跟踪是很重要的事情。理论上，**一个用户的所有请求操作都应该属于同一个会话**，而另一个用户的所有请求操作则应该属于另一个会话，二者不能混淆。会话通俗点来说，一端与服务端进行多次交互，那么它们之间交互状态转移就是会话。

在Web应用程序是使用HTTP协议传输数据的。**HTTP协议是无状态的协议。一旦数据交换完毕，客户端与服务器端的连接就会关闭，再次交换数据需要建立新的连接。这就意味着服务器无法从连接上跟踪会话**。即用户A购买了一件商品放入购物车内，当再次购买商品时服务器已经无法判断该购买行为是属于用户A的会话还是用户B的会话了。要跟踪该会话，必须引入一种机制。

Cookie就是这样的一种机制。它可以弥补HTTP协议无状态的不足。在Session出现之前，基本上所有的网站都采用Cookie来跟踪会话。

### **什么是Cookie**

Cookie意为“甜饼”，是**由W3C组织提出**，最早由Netscape社区发展的一种机制。目前Cookie已经成为标准，所有的主流浏览器如IE、Netscape、Firefox、Opera等都支持Cookie。

由于HTTP是一种无状态的协议，服务器单从网络连接上无从知道客户身份。怎么办呢？就**给客户端们颁发一个通行证吧，每人一个，无论谁访问都必须携带自己通行证。这样服务器就能从通行证上确认客户身份了。这就是Cookie的工作原理**。

Cookie实际上是一小段的文本信息。客户端请求服务器，如果服务器需要记录该用户状态，就使用response向客 户端浏览器颁发一个Cookie。客户端浏览器会把Cookie保存起来。当浏览器再请求该网站时，浏览器把请求的网址连同该Cookie一同提交给服务 器。服务器检查该Cookie，以此来辨认用户状态。服务器还可以根据需要修改Cookie的内容。

**注意：**Cookie功能需要浏览器的支持。如果浏览器不支持Cookie（如大部分手机中的浏览器）或者把Cookie禁用了，Cookie功能就会失效。不同的浏览器采用不同的方式保存Cookie。如 IE浏览器会在“C:\Documents and Settings\你的用户名\Cookies”文件夹下以文本文件形式保存，一个文本文件保存一个Cookie。

### 记录用户访问次数

Java中把Cookie封装成了javax.servlet.http.Cookie类。每个Cookie都是该Cookie类的对象。服务器通过操作Cookie类对象对客户端Cookie进行操作。通过**request.getCookie()获取客户端提交的所有Cookie**（以Cookie[]数组形式返回），**通过response.addCookie(Cookiecookie)向客户端设置Cookie。**

Cookie对象使用key-value属性对的形式保存用户状态，一个Cookie对象保存一个属性对，一个request或者response同时使用多个Cookie。因为Cookie类位于包javax.servlet.http.*下面，所以JSP中不需要import该类。

### Cookie的不可跨域名性

很多网站都会使用Cookie。例如，Google会向客户端颁发Cookie，Baidu也会向客户端颁发Cookie。那浏览器访问Google会不会也携带上Baidu颁发的Cookie呢？或者Google能不能修改Baidu颁发的Cookie呢？

答案是否定的。**Cookie具有不可跨域名性**。根据Cookie规范，浏览器访问Google只会携带Google的Cookie，而不会携带Baidu的Cookie。Google也只能操作Google的Cookie，而不能操作Baidu的Cookie。

Cookie在客户端是由浏览器来管理的。浏览器能够保证Google只会操作Google的Cookie而不会操作 Baidu的Cookie，从而保证用户的隐私安全。浏览器判断一个网站是否能操作另一个网站Cookie的依据是域名。Google与Baidu的域名 不一样，因此Google不能操作Baidu的Cookie。

### 编码

Cookie不仅可以使用ASCII字符与Unicode字符，还可以使用二进制数据。例如在Cookie中使用数字证书，提供安全度。使用二进制数据时也需要进行编码。由于浏览器每次请求服务器都会携带Cookie，因此Cookie内容不宜过多，否则影响速度。

**注：**中文与英文字符不同，**中文属于Unicode字符，在内存中占4个字符，而英文属于ASCII字符，内存中只占2个字节**。Cookie中使用Unicode字符时需要对Unicode字符进行编码，否则会乱码。[java常见编码格式](../java/java中常见编码格式分析.md)

### 设置Cookie

除了name与value之外，Cookie还具有其他几个常用的属性。每个属性对应一个getter方法与一个setter方法。Cookie类的所有属性：

| 属  性  名        | 描    述                                   |
| -------------- | ---------------------------------------- |
| String name    | 该Cookie的名称。Cookie一旦创建，名称便不可更改            |
| Object value   | 该Cookie的值。如果值为Unicode字符，需要为字符编码。如果值为二进制数据，则需要使用BASE64编码 |
| **int maxAge** | **该Cookie失效的时间，单位秒。如果为正数，则该Cookie在maxAge秒之后失效。如果为负数，该Cookie为临时Cookie，关闭浏览器即失效，浏览器也不会以任何形式保存该Cookie。如果为0，表示删除该Cookie。默认为–1** |
| boolean secure | 该Cookie是否仅被使用安全协议传输。安全协议。安全协议有HTTPS，SSL等，在网络上传输数据之前先将数据加密。默认为false |
| String path    | 该Cookie的使用路径。如果设置为“/sessionWeb/”，则只有contextPath为“/sessionWeb”的程序可以访问该Cookie。如果设置为“/”，则本域名下contextPath都可以访问该Cookie。注意最后一个字符必须为“/” |
| String domain  | 可以访问该Cookie的域名。如果设置为“.google.com”，则所有以“google.com”结尾的域名都可以访问该Cookie。注意第一个字符必须为“.” |
| String comment | 该Cookie的用处说明。浏览器显示Cookie信息的时候显示该说明       |
| int version    | 该Cookie使用的版本号。0表示遵循Netscape的Cookie规范，1表示遵循W3C的RFC 2109规范 |

### **Cookie的有效期**

Cookie的maxAge决定着Cookie的有效期，单位为秒（Second）。Cookie中通过getMaxAge()方法与setMaxAge(int maxAge)方法来读写maxAge属性。

如果maxAge属性为正数，则表示该Cookie会在maxAge秒之后自动失效。浏览器会将maxAge为正数的 Cookie持久化，即写到对应的Cookie文件中。无论客户关闭了浏览器还是电脑，只要还在maxAge秒之前，登录网站时该Cookie仍然有效。 下面代码中的Cookie信息将永远有效。

```java
Cookie cookie = new Cookie("username","helloweenvsfei");   // 新建Cookie
cookie.setMaxAge(Integer.MAX_VALUE);           // 设置生命周期为MAX_VALUE
response.addCookie(cookie);                    // 输出到客户端
```

如果maxAge为负数，则表示该Cookie仅在本浏览器窗口以及本窗口打开的子窗口内有效，关闭窗口后该 Cookie即失效。maxAge为负数的Cookie，为临时性Cookie，不会被持久化，不会被写到Cookie文件中。Cookie信息保存在浏 览器内存中，因此关闭浏览器该Cookie就消失了。Cookie默认的maxAge值为–1。

如果maxAge为0，则表示删除该Cookie。Cookie机制没有提供删除Cookie的方法，因此通过设置该Cookie即时失效实现删除Cookie的效果。失效的Cookie会被浏览器从Cookie文件或者内存中删除。

Cookie并不提供修改、删除操作。如果要修改某个Cookie，只需要新建一个同名的Cookie，添加到response中覆盖原来的Cookie。如果要删除某个Cookie，只需要新建一个同名的Cookie，并将maxAge设置为0，并添加到response中覆盖原来的Cookie。注意是0而不是负数。负数代表其他的意义。读者可以通过上例的程序进行验证，设置不同的属性。

**注意：**修改、删除Cookie时，新建的Cookie除value、maxAge之外的所有属性，例如name、path、domain等，都要与原Cookie完全一样。否则，浏览器将视为两个不同的Cookie不予覆盖，导致修改、删除失败。

### **Cookie的域名**

Cookie是不可跨域名的。域名www.google.com颁发的Cookie不会被提交到域名www.baidu.com去。这是由Cookie的隐私安全机制决定的。隐私安全机制能够禁止网站非法获取其他网站的Cookie。

如果同一个一级域名下的两个二级域名如www.helloweenvsfei.com和 images.helloweenvsfei.com也交互使用Cookie，想所有 helloweenvsfei.com名下的二级域名都可以使用该Cookie，需要设置Cookie的domain参数：

```java
Cookie cookie = new Cookie("time","20080808"); // 新建Cookie
cookie.setDomain(".helloweenvsfei.com");           // 设置域名
cookie.setPath("/");                              // 设置路径
cookie.setMaxAge(Integer.MAX_VALUE);               // 设置有效期
response.addCookie(cookie);                       // 输出到客户端
```

<br>

## Session机制

### 什么是Session

[http](./HTTP深入学习.md)是无状态协议，每次请求都是独立的线程。所以为了维护上下文信息，追踪同一个用户。

Session是另一种记录客户状态的机制，不同的是Cookie保存在客户端浏览器中，而Session保存在服务器上。客户端浏览器访问服务器的时候，服务器把客户端信息以某种形式记录在服务器上。这就是Session。客户端浏览器再次访问时只需要从该Session中查找该客户的状态就可以了。

如果说**Cookie机制是通过检查客户身上的“通行证”来确定客户身份的话，那么Session机制就是通过检查服务器上的“客户明细表”来确认客户身份。Session相当于程序在服务器上建立的一份客户档案，客户来访的时候只需要查询客户档案表就可以了。**

Session对应的类为javax.servlet.http.HttpSession类。每个来访者对应一个Session对象，所有该客户的状态信息都保存在这个Session对象里。**Session对象是在客户端第一次请求服务器的时候创建的**。

当多个客户端执行程序时，服务器会保存多个客户端的Session。获取Session的时候也不需要声明获取谁的Session。**Session机制决定了当前客户只会获取到自己的Session，而不会获取到别人的Session。各客户的Session也彼此独立，互不可见**。

提示**：Session的使用比Cookie方便，但是过多的Session存储在服务器内存中，会对服务器造成压力。**

<br>

### Session的生命周期

Session保存在服务器端。**为了获得更高的存取速度，服务器一般把Session放在内存里。每个用户都会有一个独立的Session。如果Session内容过于复杂，当大量客户访问服务器时可能会导致内存溢出。因此，Session里的信息应该尽量精简。**

**Session在用户第一次访问服务器的时候自动创建**。需要注意只有访问JSP、Servlet等程序时才会创建Session，只访问HTML、IMAGE等静态资源并不会创建Session。如果尚未生成Session，也可以使用request.getSession(true)强制生成Session。

**Session生成后，只要用户继续访问，服务器就会更新Session的最后访问时间，并维护该Session**。用户每访问服务器一次，无论是否读写Session，服务器都认为该用户的Session“活跃（active）”了一次。

由于会有越来越多的用户访问服务器，因此Session也会越来越多。**为防止内存溢出，服务器会把长时间内没有活跃的Session从内存删除。这个时间就是Session的超时时间。如果超过了超时时间没访问过服务器，Session就自动失效了。**

Session的超时时间也可以在web.xml中修改。另外，通过调用Session的invalidate()方法可以使Session失效。<br>

### **Session的常用方法**

HttpSession的常用方法

| 方  法  名                                  | 描    述                                   |
| ---------------------------------------- | ---------------------------------------- |
| void setAttribute(String attribute, Object value) | 设置Session属性。value参数可以为任何Java Object。通常为Java Bean。value信息不宜过大 |
| String getAttribute(String attribute)    | 返回Session属性                              |
| Enumeration getAttributeNames()          | 返回Session中存在的属性名                         |
| void removeAttribute(String attribute)   | 移除Session属性                              |
| String getId()                           | 返回Session的ID。该ID由服务器自动创建，不会重复            |
| long getCreationTime()                   | 返回Session的创建日期。返回类型为long，常被转化为Date类型，例如：Date createTime = new Date(session.get CreationTime()) |
| long getLastAccessedTime()               | 返回Session的最后活跃时间。返回类型为long               |
| int getMaxInactiveInterval()             | 返回Session的超时时间。单位为秒。超过该时间没有访问，服务器认为该Session失效 |
| void setMaxInactiveInterval(int second)  | 设置Session的超时时间。单位为秒                      |
| void putValue(String attribute, Object value) | 不推荐的方法。已经被setAttribute(String attribute, Object Value)替代 |
| Object getValue(String attribute)        | 不被推荐的方法。已经被getAttribute(String attr)替代   |
| boolean isNew()                          | 返回该Session是否是新创建的                        |
| void invalidate()                        | 使该Session失效                              |

Tomcat中Session的默认超时时间为20分钟。可以修改web.xml改变Session的默认超时时间。

```java
<session-config>
   <session-timeout>60</session-timeout>      <!-- 单位：分钟 -->
</session-config>
```

<br>

### Session对浏览器的要求

虽然Session保存在服务器，对客户端是透明的，它的正常运行仍然需要客户端浏览器的支持。这是因为Session 需要使用Cookie作为识别标志。HTTP协议是无状态的，Session不能依据HTTP连接来判断是否为同一客户，因此服务器向客户端浏览器发送一 个名为JSESSIONID的Cookie，它的值为该Session的id（也就是HttpSession.getId()的返回值）。Session 依据该Cookie来识别是否为同一用户。

该Cookie为服务器自动生成的，它的maxAge属性一般为–1，表示仅当前浏览器内有效，并且各浏览器窗口间不共享，关闭浏览器就会失效。

因此同一机器的两个浏览器窗口访问服务器时，会生成两个不同的Session。但是由浏览器窗口内的链接、脚本等打开的新窗口（也就是说不是双击桌面浏览器图标等打开的窗口）除外。这类子窗口会共享父窗口的Cookie，因此会共享一个Session。

注意：新开的浏览器窗口会生成新的Session，但子窗口除外。子窗口会共用父窗口的Session。例如，在链接上右击，在弹出的快捷菜单中选择“在新窗口中打开”时，子窗口便可以访问父窗口的Session。

如果客户端浏览器将Cookie功能禁用，或者不支持Cookie怎么办？例如，绝大多数的手机浏览器都不支持Cookie。Java Web提供了另一种解决方案：URL地址重写。

**URL地址重写的原理**是将该用户Session的id信息重写 到URL地址中。服务器能够解析重写后的URL获取Session的id。这样即使客户端不支持Cookie，也可以使用Session来记录用户状态。 当然也可以把该用户Session的id 弄到隐藏表单域中。

如果服务器需要**Session中禁止使用Cookie**，如修改Tomcat全局的conf/context.xml 

```xml‘
<!-- The contents of this file will be loaded for eachweb application -->
<Context cookies="false">
    <!-- ... 中间代码略 -->
</Context>
```

<br>

## tomcat session机制

### 请求过程中session操作

在请求过程中首先要解析请求中的sessionId信息，然后将sessionId存储到request的参数列表中。然后再从request获取session的时候，如果存在sessionId那么就根据Id从session池中获取session，如果sessionId不存在或者session失效，那么则新建session并且将session信息放入session池，供下次使用。

我们使用session一般这样调用request.getSession()。可以看下这方法是如何执行 请求的。

```java
	protected Session doGetSession(boolean create) {  
  
        ……  
        // 先获取所在context的manager对象  
        Manager manager = null;  
        if (context != null)  
            manager = context.getManager();  
        if (manager == null)  
            return (null);      // Sessions are not supported  
          
        //这个requestedSessionId就是从Http request中解析出来的  
        if (requestedSessionId != null) {  
            try {  
                //manager管理的session池中找相应的session对象  
                session = manager.findSession(requestedSessionId);  
            } catch (IOException e) {  
                session = null;  
            }  
            //判断session是否为空及是否过期超时  
            if ((session != null) && !session.isValid())  
                session = null;  
            if (session != null) {  
                //session对象有效，记录此次访问时间  
                session.access();  
                return (session);  
            }  
        }  
  
        // 如果参数是false，则不创建新session对象了，直接退出了  
        if (!create)  
            return (null);  
        if ((context != null) && (response != null) &&  
            context.getCookies() &&  
            response.getResponse().isCommitted()) {  
            throw new IllegalStateException  
              (sm.getString(”coyoteRequest.sessionCreateCommitted”));  
        }  
  
        // 开始创建新session对象  
        if (connector.getEmptySessionPath()   
                && isRequestedSessionIdFromCookie()) {  
            session = manager.createSession(getRequestedSessionId());  
        } else {  
            session = manager.createSession(null);  
        }  
  
        // 将新session的jsessionid写入cookie，传给browser  
        if ((session != null) && (getContext() != null)  
               && getContext().getCookies()) {  
            Cookie cookie = new Cookie(Globals.SESSION_COOKIE_NAME,  
                                       session.getIdInternal());  
            configureSessionCookie(cookie);  
            response.addCookieInternal(cookie);  
        }  
        //记录session最新访问时间  
        if (session != null) {  
            session.access();  
            return (session);  
        } else {  
            return (null);  
        }  
    }  
```

根据requestedSessionId 去查找session，找不到就创建新的session?

StandardManager的createSession方法，了解一下session的创建过程； 

```java
public Session createSession(String sessionId) {  
是个session数量控制逻辑，超过上限则抛异常退出  
    if ((maxActiveSessions >= 0) &&  
        (sessions.size() >= maxActiveSessions)) {  
        rejectedSessions++;  
        throw new IllegalStateException  
            (sm.getString(”standardManager.createSession.ise”));  
    }  
    return (super.createSession(sessionId));  
}  
```

这个最大支持session数量maxActiveSessions是可以配置的，先不管这个安全控制逻辑，看其主逻辑，即调用其基类的createSession方法； 

看下父类是怎么创建的？

```java
public Session createSession(String sessionId) {         
        // 创建一个新的StandardSession对象  
        Session session = createEmptySession();  
  
        // Initialize the properties of the new session and return it  
        session.setNew(true);  
        session.setValid(true);  
        session.setCreationTime(System.currentTimeMillis());  
        session.setMaxInactiveInterval(this.maxInactiveInterval);  
        if (sessionId == null) {  
            //设置jsessionid  
            sessionId = generateSessionId();  
        }  
        session.setId(sessionId);  
        sessionCounter++;  
        return (session);  
    }  
```

那么requestedSessionId 怎么获取的呢？

```java
protected synchronized String generateSessionId() {  
  
        byte random[] = new byte[16];  
        String jvmRoute = getJvmRoute();  
        String result = null;  
  
        // Render the result as a String of hexadecimal digits  
        StringBuffer buffer = new StringBuffer();  
        do {  
            int resultLenBytes = 0;  
            if (result != null) {  
                buffer = new StringBuffer();  
                duplicates++;  
            }  
  
            while (resultLenBytes < this.sessionIdLength) {  
                getRandomBytes(random);  
                random = getDigest().digest(random);  
                for (int j = 0;  
                j < random.length && resultLenBytes < this.sessionIdLength;  
                j++) {  
                    byte b1 = (byte) ((random[j] & 0xf0) >> 4);  
                    byte b2 = (byte) (random[j] & 0x0f);  
                    if (b1 < 10)  
                        buffer.append((char) (‘0’ + b1));  
                    else  
                        buffer.append((char) (‘A’ + (b1 - 10)));  
                    if (b2 < 10)  
                        buffer.append((char) (‘0’ + b2));  
                    else  
                        buffer.append((char) (‘A’ + (b2 - 10)));  
                    resultLenBytes++;  
                }  
            }  
            if (jvmRoute != null) {  
                buffer.append(’.’).append(jvmRoute);  
            }  
            result = buffer.toString();  
        //注意这个do…while结构  
        } while (sessions.containsKey(result));  
        return (result);  
    }  
```

创建jsessionid的方式是由tomcat内置的加密算法算出一个随机的jsessionid，如果此jsessionid已经存在，则重新计算一个新的，直到确保现在计算的jsessionid唯一。 

<br>

### 注销session

- 主动注销
- 超时注销

主动注销时，是调用标准的servlet接口： session.invalidate()：tomcat提供的标准session实现(StandardSession) ：

```java
public void invalidate() {  
        if (!isValidInternal())  
            throw new IllegalStateException  
                (sm.getString(”standardSession.invalidate.ise”));  
        // 明显的注销方法  
        expire();  
    }  
```

超时注销时，Tomcat定义了一个最大空闲超时时间，也就是说当session没有被操作超过这个最大空闲时间时间时，再次操作这个session，这个session就会触发expire。 
这个方法封装在StandardSession中的isValid()方法内，这个方法在获取这个request请求对应的session对象时调用，可以参看上面说的创建session环节。也就是说，获取session的逻辑是，先从manager控制的session池中获取对应jsessionid的session对象，如果获取到，就再判断是否超时，如果超时，就expire这个session了。 
看一下tomcat提供的标准session实现(StandardSession) ：

```java
public boolean isValid() {  
        ……  
        //这就是判断距离上次访问是否超时的过程  
        if (maxInactiveInterval >= 0) {   
            long timeNow = System.currentTimeMillis();  
            int timeIdle = (int) ((timeNow - thisAccessedTime) / 1000L);  
            if (timeIdle >= maxInactiveInterval) {  
                expire(true);  
            }  
        }  
        return (this.isValid);  
    }  
```

可以发现主动、被动撤销都调用了expire方法：

```java
public void expire(boolean notify) {   
  
        synchronized (this) {  
            ……  
            //设立标志位  
            setValid(false);  
  
            //计算一些统计值，例如此manager下所有session平均存活时间等  
            long timeNow = System.currentTimeMillis();  
            int timeAlive = (int) ((timeNow - creationTime)/1000);  
            synchronized (manager) {  
                if (timeAlive > manager.getSessionMaxAliveTime()) {  
                    manager.setSessionMaxAliveTime(timeAlive);  
                }  
                int numExpired = manager.getExpiredSessions();  
                numExpired++;  
                manager.setExpiredSessions(numExpired);  
                int average = manager.getSessionAverageAliveTime();  
                average = ((average * (numExpired-1)) + timeAlive)/numExpired;  
                manager.setSessionAverageAliveTime(average);  
            }  
  
            // 将此session从manager对象的session池中删除  
            manager.remove(this);  
            ……  
        }  
    }  
```

<br>

## Tomcat Session的管理机制

 实现包的路径是：org.apache.catalina.session，

tomcat对外提供session调用的接口不在这个实现包里，对外接口是在包javax.servlet.http下的HttpSession。

而实现包里的StandardSession是tomcat提供的标准实现，当然对外tomcat不希望用户直接操作StandardSession，而是提供了一个StandardSessionFacade类，tomcat容器里具体操作session的组件是servlet，

而servlet操作session是通过StandardSessionFacade进行的，这样就可以防止程序员直接操作StandardSession所带来的安全问题。

session包下各类含义：

-  Manager：定义了关联到某一个容器的用来管理session池的基本接口。这是catalina包下的。
-  ManagerBase：实现了Manager接口，该类提供了Session管理器的常见功能的实现。
-  StandardManager：继承自ManagerBase,tomcat的默认Session管理器（不指定配置，默认使用这个），是tomcat处理session的非集群实现（也就说是单机版的），tomcat关闭时，内存session信息会持久化到磁盘保存为SESSION.ser,再次启动时恢复。
-  PersistentManagerBase：继承自ManagerBase，实现了和定义了session管理器持久化的基础功能。
-  PersistentManager：继承自PersistentManagerBase，主要实现的功能是会把空闲的会话对象（通过设定超时时间）交换到磁盘上。
-  StoreBase:存放持久化有关store接口的基本实现。接口Store及其实例是为session管理器提供了一套存储策略，store定义了基本的接口，而StoreBase提供了基本的实现。其中FileStore类实现的策略是将session存储在以setDirectory（）指定目录并以.session结尾的文件中的。JDBCStore类是将Session通过JDBC存入数据库中，因此需要使用JDBCStore，需要分别调用setDriverName()方法和setConnectionURL()方法来设置驱动程序名称和连接URL。
-  StandardSession :tomcat的session实现类。

其实session管理器或者Store存储策略，只要实现了相关的接口，都是可以自定义的。

比如SessionConfig这个类就定义了一些session的设 置信息。Session在cookie中的名字是JSESSION. Session通过URL重写的方式放在path里时，键值的名字是jsessionid。还有一点在tomcat-7 SessionIdGenerator当中指定 sessionId默认长度是16个字节。

<br>

具体参考Tomcat Jar： org\apache\tomcat\embed\tomcat-embed-core\8.5.31