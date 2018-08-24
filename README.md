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
用这个token从缓存中取用户信息，将它传给controller方法。如果用户信息存在，延长cookie有效期。（为什么要延长？）
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
[Spring Boot 进行Bean Validate和Method Validate](https://blog.csdn.net/baidu_35776955/article/details/79551459)  
全局异常处理，@Valid校验 抛BindException  

# 秒杀实现
## 商品详情
java.util.Date 
getTime() 毫秒
用goodsid拿出商品信息，判断是否已经开始秒杀，设置秒杀状态和倒计时时间，开始秒杀倒计时为0，结束为-1。  
## 秒杀
1. 判断库存：先根据商品id拿到商品信息，来判断库存，没库存了返回已秒杀完毕。  
2. 是否重复秒杀：有库存根据userid和goodsid从秒杀订单表里取数据，判断是否已经下了单秒杀过这个商品了，已下单返回不能重复秒杀。  
3. 事务，减库存、生成订单、写入秒杀订单

一般在service中不直接调用其他功能的dao而是要调用service，因为有可能在对应的service做了缓存或者其他操作。
# jmeter 压测
redis压测工具，redis-benchmark  
[springboot 打war包](https://blog.csdn.net/qq_28988969/article/details/78135768)  
[聊聊QPS/TPS/并发量/系统吞吐量的概念](https://blog.csdn.net/cainiao_user/article/details/77146049)
QPS:服务器每秒处理完多少个请求  
吞吐量（QPS）= 并发量/平均响应时间  
[QPS 和并发：如何衡量服务器端性能](https://blog.csdn.net/leyangjun/article/details/64131491)  
linux 的top 监控服务器资源内存cpu等  

# 页面级优化
redis缓存、静态化分离  
高并发瓶颈在数据库，可用缓存解决。  
用户发起请求，先通过页面静态化，从浏览器获取页面缓存。部署cdn节点，让请求首先访问cdn节点，页面缓存，对象缓存，最后数据库。一步一步通过缓存，削减到数据库的请求量。通过缓存可能会引起数据不一致，只能尽量平衡，在满足数据一致的前提下进行缓存。  
cdn：内容分发网络，将数据缓存到各个节点上，有请求将请求重新导向到最近的服务节点。  
## 页面缓存
商品列表和商品详情页 qps提高100%，mysql负载降低60%  
  
将模板手动渲染成html后存进redis中  
1. 每次请求都先判断缓存中是否有该页面
2. 有就直接返回
3. 没有的话手动渲染后存进缓存中
4. 一般缓存过期时间不能设置太长，以防数据更新不及时

ThymeleafViewResolver 视图解析器获得视图引擎后来渲染模板 
@RequestMapping(value="/to_list", produces="text/html") [produces属性含义](https://blog.csdn.net/lzwglory/article/details/17252099)    
表示方法将生产html格式的数据，此时根据请求头中的Accept进行匹配，如请求头“Accept:text/html”时即可匹配。    
@ResponseBody 一般是返回json时使用，但它的确切含义是将返回的对象写进响应的body中，
在使用此注解之后不会再试图走处理器，而是直接将数据写入到输入流中，他的效果等同于通过response对象输出指定格式的数据。  
@RequestMapping(value="/to_detail2/{goodsId}",produces="text/html")  这类url同样可以缓存页面，不过key需要对应goodsId  

## 对象缓存  
缓存秒杀订单对象，不设过期时间    
同样用redis缓存，下单创建订单后缓存。秒杀判断有无重复秒杀时拿的是缓存里的不是从数据库拿，这样降低了数据库的压力。  
  
如果有更新是需要先更新数据库再让缓存失效。反过来不行。  
https://blog.csdn.net/lc0817/article/details/52089473
> 假设有两个并发操作，一个是更新操作，另一个是查询操作，更新操作删除缓存后，查询操作没有命中缓存，先把老数据读出来后放到缓存中，然后更新操作更新了数据库。于是，在缓存中的数据还是老的数据，导致缓存中的数据是脏的，而且还一直这样脏下去了。  
  
更新时新new一个bean，更新哪个字段加哪个  ？

## 页面静态化
页面静态化，AngularJS，vue.js  
在浏览器缓存，springboot需要配置对静态文件的处理。  
没有配会想服务器发起一次请求，服务器收到请求确定页面没有改变就返回304让浏览器使用缓存。  
Cache-Control：浏览器缓存该资源多长时间  
  
响应码304：服务端收到参数检查页面有没有变化，没有变化返回304，客户端可直接用浏览器缓存。

这样只有json跟服务器交互了，提高了响应速度？

Pragma  
Expire  
Cache-Control  
## 卖超问题
```update miaosha_goods set stock_count = stock_count - 1 where goods_id = #{goodsId} and stock_count > 0```  
数据库会自动加锁确保只有一个线程执行该sql，保证商品不卖超 ?没有更新0条就回滚事务的逻辑啊
  
一个用户同时发两个请求，会都通过是否已秒杀到商品的校验，所以会秒杀到两个商品。  
通过数据库解决，在秒杀订单表指定字段加上唯一索引防止重复购买。  
## 静态资源优化
js/css压缩（去空格等），减少流量。库一般都有提供压缩版的文件。  
多个js/css组合，减少连接数，三次握手，多个连接会导致刚开网页时很慢  
Tengine  
webpack 打包  
   
# 其他
[一张图搞定OAuth2.0](https://www.cnblogs.com/flashsun/p/7424071.html)
[springboot(十四)：springboot整合shiro-登录认证和权](https://yq.aliyun.com/articles/385516)  
[HTTP幂等性及GET、POST、PUT、DELETE的区别](https://blog.csdn.net/zjkc050818/article/details/78799386)
get post 传参的大小是浏览器厂商实现时自己限制的，http协议并没有规定参数大小。他们的主要区别是幂等性。幂等性属于语义范畴？  

1. 幂等性，n次请求和1次请求产生的副作用相同，get没有副作用，delete调用多次都是删除同一个资源，put是创建或更新同一uri
2. 一般显式的id不用自增长，因为很有可能会被遍历出来所有信息，snowflake算法？
3. com.alibaba.fastjson java类和json的互转