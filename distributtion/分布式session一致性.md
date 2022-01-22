# 分布式系统session一致性的问题

参考链接：https://yq.aliyun.com/ziliao/105539

<br>

## 场景

在多台后台服务器的环境下，我们为了确保一个客户只和一台服务器通信，我们势必使用长连接。使用什么方式来实现这种连接呢，常见的有使用nginx自带的ip_hash来做，我想这绝对不是一个好的办法，如果前端是[CDN](https://www.aliyun.com/product/cdn)，或者说一个局域网的客户同时访问服务器，导致出现服务器分配不均衡，以及不能保证每次访问都粘滞在同一台服务器。如果基于cookie会是一种什么情形，想想看, 每台电脑都会有不同的cookie，在保持长连接的同时还保证了服务器的压力均衡。

<br>

**问题分析：**

1. 一开始请求过来，没有带session信息，jvm_route就根据round robin的方法，发到一台tomcat上面。


2. tomcat添加上session 信息，并返回给客户。
3. 用户再此请求，jvm_route看到session中有后端服务器的名称，它就把请求转到对应的服务器上。

暂时jvm_route模块还不支持默认fair的模式。jvm_route的工作模式和fair是冲突的。对于某个特定用户，当一直为他服务的 tomcat宕机后，默认情况下它会重试max_fails的次数，如果还是失败，就重新启用round robin的方式，而这种情况下就会导致用户的session丢失。

总的说来，jvm_route是通过session_cookie这种方式来实现session粘性，将特定会话附属到特定tomcat上,从而解决session不同步问题，但无法解决宕机后会话转移问题。
假如没有这个jvm_route,用户再请求的时候，由于没有session信息，nignx就会再次随机的发送请求到后端的tomcat服务器，这种情况，对于普通的页面访问是没有问题的。对于带有登录验证信息的请求，其结果就是永远登录不了应用服务器。
这个模块通过session cookie的方式来获取session粘性。如果在cookie和url中并没有session，则这只是个简单的round-robin 负载均衡。

<br>

**解决方案**

### ip_hash（不推荐使用）

nginx中的ip_hash技术能够将某个ip的请求定向到同一台后端，这样一来这个ip下的某个客户端和某个后端就能建立起稳固的session，ip_hash是在upstream配置中定义的： 

     upstream backend {   
        server 192.168.12.10:8080 ;   
        server 192.168.12.11:9090 ;   
        ip_hash;   
    }

不推荐使用的原因如下：

  1 nginx不是最前端的服务器。

   ip_hash要求nginx一定是最前端的服务器，否则nginx得不到正确ip，就不能根据ip作hash。譬如使用的是squid为最前端，那么nginx取ip时只能得到squid的服务器ip地址，用这个地址来作分流是肯定错乱的。

​       2 nginx的后端还有其它方式的负载均衡。

   假如nginx后端又有其它负载均衡，将请求又通过另外的方式分流了，那么某个客户端的请求肯定不能定位到同一台session应用服务器上。

   3 多个外网出口。

​    很多公司上网有多个出口，多个ip地址，用户访问互联网时候自动切换ip。而且这种情况不在少数。使用 ip_hash 的话对这种情况的用户无效，无法将某个用户绑定在固定的tomcat上 。

<br><br>

### nginx_upstream_jvm_route(nginx扩展，推荐使用)

nginx_upstream_jvm_route 是一个nginx的扩展模块，用来实现基于 Cookie 的 Session Sticky 的功能。

简单来说，它是基于cookie中的JSESSIONID来决定将请求发送给后端的哪个server，nginx_upstream_jvm_route会在用户第一次请求后端server时，将响应的server标识绑定到cookie中的JSESSIONID中，从而当用户发起下一次请求时，nginx会根据JSESSIONID来决定由哪个后端server来处理。

1 nginx_upstream_jvm_route安装

下载地址(svn)：http://nginx-upstream-jvm-route.googlecode.com/svn/trunk/

假设nginx_upstream_jvm_route下载后的路径为/usr/local/nginx_upstream_jvm_route，

(1)进入nginx源码路径

patch -p0 < /usr/local/nginx_upstream_jvm_route/jvm_route.patch

(2)./configure  --with-http_stub_status_module --with-http_ssl_module --prefix=/usr/local/nginx --with-pcre=/usr/local/pcre-8.33 --add-module=/usr/local/nginx_upstream_jvm_route

(3)make & make install

<br>

2 nginx配置

        upstream  tomcats_jvm_route  
        {  
             # ip_hash;   
              server   192.168.33.10:8090 srun_id=tomcat01;   
              server   192.168.33.11:8090 srun_id=tomcat02;  
              jvm_route $cookie_JSESSIONID|sessionid reverse;  
        } 

 3 tomcat配置

修改192.168.33.10:8090tomcat的server.xml，

将 

<Engine name="Catalina" defaultHost="localhost" > 

修改为：

<Engine name="Catalina" defaultHost="localhost" jvmRoute="tomcat01"> 

 同理，在192.168.33.11:8090server.xml中增加jvmRoute="tomcat02"。

4 测试

启动tomcat和nginx，访问nginx代理，使用Google浏览器，F12，查看cookie中的JSESSIONID，
形如：ABCD123456OIUH897SDFSDF.tomcat01 ，刷新也不会变化

<br>

### 基于cookie的Nginx Sticky模块





