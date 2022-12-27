---
title: JWT 工具类封装
abbrlink: 64951
date: 2022-09-05 19:04:10
description: 关于 JWT 的介绍包括简介、原理、使用场景等网上有很多文章写的都非常好，本文不会过多赘述，主要记录 JWT 工具类的封装。
hide: true
categories:
 - 代码段记录
---

# JWT

JWT（Json Web Token）是一种用于用户认证的技术，前端携带 JWT 访问后端服务器，后端服务器可以解析 JWT 判断是否由本服务器签发，以及解析出一些简单的数据后可以拿到本次请求的用户。引入 JWT 主要为了解决传统 session 验证的弊端，session 认证的弊端：

1. Session 保存在服务器中，用户数增加对服务器开销造成一定压力。
2. Session 保存在服务器物理内存中，对分布式不友好。
3. 依赖 Cookie，对于非浏览器的客户端、手机移动端等不适用。
4. 客户端 Cookie 泄漏会导致服务器不安全。
5. 由于依赖 Cookie，所以无法跨域。

JWT 的优势：

1. 简洁、数据量小、传输速度快
2. 存储在客户端，原则上是跨语言的，支持任何 web 形式。
3. 不依赖 Cookie 和 Session，对分布式友好。
4. 容易跨域，对单点登录友好。
5. 对手机移动端适用。

JWT 认证流程图：

![JWT认证流程](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/JWT认证流程.png)

## JWT 结构

JWT 分成三个部分，每个部分都是一个字符串，中间由 `.` 隔开。

- Header：头部，标记加密的算法
- Payload：负载，存放具体数据
- Signature：签名，由 “Header + Payload + 服务器本地密钥” 经 MD5 加密后的值。

### Header

Header 是一个描述 JWT 元数据的 JSON 对象，alg 属性表示签名使用的算法，默认为 HMAC SHA256（写为HS256）；typ 属性表示令牌的类型，JWT 令牌统一写为 JWT。最后，使用 Base64 URL 算法将上述 JSON 对象转换为字符串保存。一般是下面这样：

```json
{
  "typ": "JWT",
  "alg": "HS256"
}
```

### Payload

有效载荷部分，是 JWT 的主体内容部分，也是一个 JSON 对象，包含需要传递的数据，可以存放自定义数据。JWT 指定七个默认字段供选择

1. iss (issuer)：签发人
2. exp (expiration time)：过期时间
3. sub (subject)：主题
4. aud (audience)：受众
5. nbf (Not Before)：生效时间
6. iat (Issued At)：签发时间
7. jti (JWT ID)：编号

注意：此部分内容未加密，不能存放敏感信息。

### Signature

signature 是签证信息，该签证信息是通过 `header` 和 `payload`，加上 `secret`，通过算法加密生成。

# JWTUtil

```xml
<!-- 引入 JWT 依赖 -->
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>4.0.0</version>
</dependency>
```

```java
import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTCreator;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.exceptions.JWTDecodeException;
import com.auth0.jwt.interfaces.DecodedJWT;
import org.springframework.stereotype.Component;

import java.util.Calendar;
import java.util.Map;

@Component
public class JWTUtil {

    // 可从 application.yml 中获取
    private final static String SECRET = "maxiaorui";

    /**
     * 生成用户 Token
     *
     * @param map     存放在 token 的数据
     * @param expires 过期时间（单位：秒）
     * @return token
     */
    public static String generateUserToken(Map<String, String> map, Integer expires) {
        final JWTCreator.Builder builder = JWT.create();

        // payload
        map.forEach(builder::withClaim);

        // 指定过期时间
        Calendar expiresAt = Calendar.getInstance();
        expiresAt.add(Calendar.SECOND, expires);
        builder.withExpiresAt(expiresAt.getTime());

//        builder.withIssuer("issuer");                       // 签发人
//        builder.withSubject("subject");                     // 主题
//        builder.withAudience("audience1", "audience2");     // 受众
//        builder.withNotBefore(new Date());                  // 生效时间
//        builder.withIssuedAt(new Date());                   // 签发时间
//        builder.withJWTId("jti");                           // 编号

        // 生成原始 Token，此处可对 payload 数据做混淆
        return builder.sign(Algorithm.HMAC256(SECRET));
    }
    
    /**
     * 校验用户 Token
     *
     * @param token 用户 Token
     * @return DecodedJWT
     */
    public static DecodedJWT verify(String token) {
        if (token == null || "".equals(token))
            throw new JWTDecodeException("Token 无效");

        // 如果在生成 Token 的时候做了混淆此处应该解析混淆
        return JWT.require(Algorithm.HMAC256(SECRET)).build().verify(token);
    }
}
```

从 DecodedJWT 中解析 payload：`DecodedJWT.getClaims();`

解析可能出现的异常：

- JWTDecodeException：header、payload 被修改会出现的异常
- SignatureVerificationException：签名不匹配异常
- TokenExpiredException：令牌过期异常
- AlgorithmMismatchException：算法不匹配异常

---

说句题外话，有没有发现 JWT 的前面总是会加 `Bearer` 这个单词？？？

那么加了能干嘛呢，不加行不行呢？？

别问，问就是规范，至于什么规范？[谷歌](https://www.google.com/)、[必应](https://www.bing.com/)、[百度](https://www.baidu.com/)

> **前人栽树：**
> - [一篇文章告诉你JWT的实现原理](https://cnodejs.org/topic/5b0c4a7b8a4f51e140d942fc)
> - [JWT分布式场景应用解析 ](https://www.cnblogs.com/johnvwan/p/15557287.html)