# 分布式session  
为什么需要分布式session
因为可能不只一台服务器，有多台服务器提供秒杀功能，用户登上A，session就记录在A上，
但如果再次访问被分流到另一台服务器B上，在B中并没有该用户的session，用户通过所持有的sessionid在B中找不到对应的session。
所以为了解决这个问题，需要同步session，主流web容器一般都有提供同步的功能（？）但是同步session性能太差。  
一般是用一个登录令牌token从数据库拿到登录信息，以此识别用户是否已登录。
## 实现  
登录验证成功后生成一个uuid作为token，将它当做key把用户信息存在redis中，然后将这个token放在cookie中返回给客户端。  
客户端请求时就会附带上这个cookie。  
然后在请求进入controller前先处理一下请求，从url参数或者request头中取出cookie中的token。  
用这个token从redis中取用户信息，将它传给controller方法。  
## 关于cookie
### maxAge
cookie.setMaxAge()
1. 负值，存在内存中，关闭浏览器cookie失效  
2. 0，删除cookie
3. 正数，存活时间，单位为秒  
  
https://www.cnblogs.com/keyi/p/6122845.html  
删除时需要设置path，domain让客户端知道是要删哪个cookie？
### path
cookie.setPath()
在指定路径的时候，凡是来自同一服务器，URL里有相同路径的所有WEB页面都可以共享cookies。  
localhost:8080/test/aa  
localhost:8080/test/bb  
localhost:8080/test/bb/c  
path设为'/'则改cookie被所有'localhost:8080/test/'路径下的web页面共享  
path设为'/bb'则改cookie被所有'localhost:8080/test/bb'路径下的web页面共享  

默认只有设置该cookie的路径及其子路径可以访问


### domain

# md5
两次加密，客户端加密，http是明文传输的，然后服务端加密一次，防止数据库信息泄露时别人根据客户端的salt反推出密码。

# JSR 303 数据校验
[JSR 303 - Bean Validation 介绍及最佳实践](https://www.ibm.com/developerworks/cn/java/j-lo-jsr303/index.html)

