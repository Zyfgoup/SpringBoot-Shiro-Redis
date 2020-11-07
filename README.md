原项目地址：https://github.com/xuyulong2017/my-java-demo/tree/master/spring-boot-shiro-demo
#### 项目介绍:
并无做修改 只是写一下自己的理解

###大体流程
从ShiroConfig进行分析
1.shiro-aop的支持  因为使用到权限拦截的注解 所以需要开启这个
2.基础的Shiro配置，配置Filter拦截相对应的请求地址
3.SecurityManager的配置 注入SeesionManager CacheManager 以及自定义的Realm(处理认证和授权)
因为是使用Redis来存储session和cache 所以肯定都是要使用到Redis相关的实现类
4.shiroRealm 返回自定义的Realm即可
5.凭证匹配器，设置相对应的加密方式和盐、次数等 然后在realm的认证中通过传入的密码和盐 然后进行加密与数据库中加密后的密码做比对
6.redisManager 就是设置相对应的redis的一些参数 连接端口号 密码 过期时间等
7.cacheManager 存到redis中 所以要使用RedisCacheManager 然后注入redisManager 可以配置缓存头的字段
并且关键的是要设置与放在session里的实体类有相对应的字段 一般为用户id(主键)即可
这一点我也不太懂 通过debug可以看到在授权方式返回后 是要获取到Principal（为当前用户的UserEntity的实例）
然后作为作为key和获取到的角色权限作为key-value值存到cache中 所以需要UserId与key中的一个键对应起来吧
8.sessionId生成器  根据方法生成带有设置好前缀的sessionId
9.redisSessionDAO 操作redis存入session  注入redisManager sessionId生成器 定义前缀、过期时间等
10.sessionManager 与3相关的  这里注入相对应的redisSessionDAO即可
11.根据3、7、10 就完成了基本的shiro的cache redis的配置 然后3中注入了6返回的redisManager 然后10中使用到了RedisSessionDAO

12.编写相对应的Controller层（自定义Realm省略了） 登录时使用Subject.login进行登录 然后就会执行Realm的认证方法
13.认证成功后 session会存到Redis中
14.当访问某个地址时 如果有权限拦截或者角色拦截 就会看有没有cache 如果没有则执行授权方法 获取用户相对应的角色、权限存到cache中
15.当更新权限时需要把cache删除

 

### 测试
使用postman 
先访问http:localhost:8764/userLogin/login 传入json 
{
    "username":"admin",
    "password":"123456"
}
然后可以看到返回回来的jsseionId 
然后可以看到redis中已经存有seesionid了

然后访问http:localhost:8764/menu/getUserInfoList
但是得带上 Authorization:jssesionid去访问
然后就能拿到数据了
当第一次访问后 存在redis中的cache就会当前登录用户相对应的权限

#####当用户访问某个地址时会有相对应的权限拦截 会去cache找是否有相对应的权限 如果没有抛出异常

**如果是没有cache则执行对应的Realm中的授权的方法去获取对应的权限存到cache中**

当访问不是对应权限的 则会返回权限不足

当访问添加权限的地址时  则可以通过添加相对应的权限后 将redis的cache删除
然后重新访问一个有权限的地址后 就会把新的cache存在redis上