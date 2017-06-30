### 常用的认证

#### HTTP Basic Auth

* 每次请求时都提供用户名和密码，是最简单的认证方式
* 有把用户名密码暴露给第三方的风险，因此生产环境下越来越少地被使用

#### OAuth（开放授权）

* 允许第三方应用访问该用户在某一web服务器上存储的私密的资源，无需将用户名密码提供给第三方
* 用户提供一个令牌而非用户名密码来访问特定服务提供者的数据

![](http://images2015.cnblogs.com/blog/34831/201606/34831-20160622150107172-139099471.png)

* 适用于个人消费类的互联网产品，如社交App（微信），但不适合拥有自有认证权限管理的企业应用

#### Cookie Auth & Token Auth

![](http://images2015.cnblogs.com/blog/34831/201606/34831-20160622150124531-1416052185.png)

##### Token Auth的优点

* **支持跨域访问：**Cookie不支持跨域访问，当用户认证信息通过HTTP头传输时Token允许跨域访问
* **无状态：**Token自身包含了所有登录用户的信息，只需要在客户端存储相关信息而不需要在服务端存储
* **更适用于CDN（内容分发网络）：**可以通过内容分发网络请求服务端的所有资料
* **去耦：**不需要绑定到特定的身份验证方案，可在任何地方生成Token
* **更适用于移动平台：**原生平台不支持cookie，因此适合采用Token
* **无需考虑CSRF（跨站请求伪造）：**因不依赖cookie，因此无需考虑该安全问题
* **性能：**一次网络往返时间（通过数据库查询session信息）比做一次Token验证和解析要费时的多
* **标准化库：**可采用JWT标准的库，生态支持良好

#### 基于jwt的Token认证机制实现

##### JWT 的组成

Token是一个字符串，由三部分组成：头部、载荷、签名

* **载荷(Payload)：**

```json
{
  "iss": "Online JWT Builder", 
  "iat": 1416797419,
  "exp": 1448333419,
  "aud": "www.example.com",
  "sub": "jrocket@example.com",
  "GivenName": "Johnny",
  "Surname": "Rocket",
  "Email": "jrocket@example.com",
  "Role": [ "Manager", "Project Administrator" ]
}
```

* * **iss：**JWT签发者，可选
* * **sub：**JWT面向的用户，可选
* * **aud：**接收该JWT的一方，可选
* * **exp（expires）：**过期时间，时间戳，可选
* * **iat（issued at）：**签发时间，可选

将上面的json进行base64编码得到一下字符串，称为Payload：

```json
eyJpc3MiOiJKb2huIFd1IEpXVCIsImlhdCI6MTQ0MTU5MzUwMiwiZXhwIjoxNDQxNTk0NzIyLCJhdWQiOiJ3d3cuZXhhbXBsZS5jb20iLCJzdWIiOiJqcm9ja2V0QGV4YW1wbGUuY29tIiwiZnJvbV91c2VyIjoiQiIsInRhcmdldF91c2VyIjoiQSJ9
```

> Base64是一种基于64个可打印字符来表示二进制数据的表示方法。由于2的6次方等于64，所以每6个比特为一个单元，对应某个可打印字符。三个字节有24个比特，对应于4个Base64单元，即3个字节需要用4个可打印字符来表示

* **头部(Header)：**描述该jwt最基本的信息

```json
{
	"typ": "JWT",
	"alg": "HS256"
}
```

* * **alg：**签名算法（必选，其他都是可选）
  * **typ:**  类型 （如果是 JWT 那么就带有一个值 `JWT`）
  * **kid: **  密钥 ID
  * **cty:**  内容类型
  * **jku:**  JWK 指定 URL
  * **jwk:**  JSON 网络值
  * **x5u:** X.509 URL
  * **x5c: ** X.509 证书链
  * **x5t:**  X.509 证书 SHA-1 指纹
  * **x5t#S256:**  X.509 证书 SHA-256 指纹
  * **crit:** 临界值


将该头部也进行base64转换，得到如下字符串：

```json
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
```

* **签名（Signature）：**

将上面的两个编码后的字符串都用句号.连接在一起（Header在前），就形成了:

```json
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJKb2huIFd1IEpXVCIsImlhdCI6MTQ0MTU5MzUwMiwiZXhwIjoxNDQxNTk0NzIyLCJhdWQiOiJ3d3cuZXhhbXBsZS5jb20iLCJzdWIiOiJqcm9ja2V0QGV4YW1wbGUuY29tIiwiZnJvbV91c2VyIjoiQiIsInRhcmdldF91c2VyIjoiQSJ9
```

最后，将上面拼接完的字符串用HS256算法进行加密。在加密的时候，我们还需要提供一个密钥（secret）。比如用mystar作为密钥，加密后的内容:

```
rSWamyAYwuHCo7IFAgd1oRpSP7nzL7BF5t7ItqpKViM
```

最终的Token：

```json
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJKb2huIFd1IEpXVCIsImlhdCI6MTQ0MTU5MzUwMiwiZXhwIjoxNDQxNTk0NzIyLCJhdWQiOiJ3d3cuZXhhbXBsZS5jb20iLCJzdWIiOiJqcm9ja2V0QGV4YW1wbGUuY29tIiwiZnJvbV91c2VyIjoiQiIsInRhcmdldF91c2VyIjoiQSJ9.rSWamyAYwuHCo7IFAgd1oRpSP7nzL7BF5t7ItqpKViM
```

* **Token认证过程：**

![](http://images2015.cnblogs.com/blog/34831/201606/34831-20160622152259735-1969767936.jpg)

- * 第一次登录，用户从浏览器输入用户名/密码，提交后到服务器的登录处理的Login Action层
- * Login Action调用认证服务进行用户名密码认证，如果认证通过，Login Action层调用用户信息服务获取用户信息（包括完整的用户信息及对应权限信息）

- * 返回用户信息后，Login Action从配置文件中获取Token签名生成的秘钥信息，进行Token的生成；

- * 生成Token的过程中可以调用第三方的JWT Lib生成签名后的JWT数据

- * 完成JWT数据签名后，将其设置到COOKIE对象中，并重定向到首页，完成登录过程

**请求认证**

基于Token的认证机制会在每一次请求中都带上完成签名的Token信息，这个Token信息可能在COOKIE
中，也可能在HTTP的Authorization头中

![](http://images2015.cnblogs.com/blog/34831/201606/34831-20160622152344250-1238130627.jpg)

- 客户端（APP客户端或浏览器）通过GET或POST请求访问资源
- 认证服务作为一个Middleware HOOK 对请求进行拦截，首先在cookie中查找Token信息，如果没有找到，则在HTTP Authorization Head中查找
- 如果找到Token信息，则根据配置文件中的签名加密秘钥调用JWT Lib对Token信息进行`解密`和`解码`
- 完成解码并验证签名通过后，对Token中的exp、nbf、aud等信息进行验证
- 全部通过后，根据获取的用户的角色权限信息，进行对请求的资源的权限逻辑判断
- 如果权限逻辑判断通过则通过Response返回给客户端


##### 使用Token Oauth 需要注意的地方

- 一个Token就是一些信息的集合，因此可以在Token中包含足够多的信息，以便在后续请求中减少查询数据库的几率，其中的机密信息应加密后再存入
- 服务端需要对cookie和HTTP Authrorization Header进行Token信息的检查，因此可以用一套token认证代码来面对浏览器类客户端和非浏览器类客户端
- 因为token是被签名的，所以我们可以认为一个可以解码认证通过的token是由我们系统发放的，其中带的信息是合法有效的

#### 通过JWT登出

因为没有 session 数据存储在服务端所以不能再通过破坏 session 来注销用户。因此登出成为了客户端的职责 - 一旦客户丢失了令牌不能再被授权，就可以被认为是登出了