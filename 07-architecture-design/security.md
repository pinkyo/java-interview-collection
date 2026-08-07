# 系统安全设计

---

## 一、认证与授权

### 1.1 OAuth 2.0 授权流程

```
        用户
         │
    ┌────┴────┐          ┌─────────┐          ┌─────────┐
    │         │ 请求授权  │         │  验证     │         │
    │  前端    │────────►│ 认证服务  │◄────────│ 资源服务  │
    │         │ 返回token │         │  token   │         │
    └─────────┘          └─────────┘          └─────────┘
```

**四种授权模式：**

| 模式 | 适用场景 |
|------|---------|
| 授权码（Authorization Code） | 有后端的 Web 应用（最安全） |
| 密码（Password） | 自家 App |
| 客户端凭证（Client Credentials） | 服务间通信 |
| 隐式（Implicit） | 纯前端应用（已不推荐，用 PKCE 取代） |

### 1.2 JWT（JSON Web Token）

```
Header.Payload.Signature
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMifQ.signature

Header:  {"alg": "HS256", "typ": "JWT"}
Payload: {"sub": "123", "name": "zhang", "exp": 1620000000}
Signature: HMAC-SHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

**追问：JWT 的优缺点？**

优点：无状态（服务端不存 Session）、跨域友好、适合分布式
缺点：无法主动失效（需配合黑名单）、Payload 不加密（仅 base64 编码）

### 1.3 SSO 单点登录

```
应用A ──未登录──► SSO认证中心 ──登录──► 返回Token
应用B ──未登录──► SSO认证中心 ──已有Session──► 返回Token
```

**追问：Session 和 JWT 的区别？**

| 对比 | Session | JWT |
|------|---------|-----|
| 存储位置 | 服务端 | 客户端 |
| 扩展性 | 需 Redis 共享 | 天然支持 |
| 安全性 | Cookie HttpOnly | 需注意泄露 |
| 失效 | 服务端删除 | 无法主动失效（需黑名单） |
| 状态 | 有状态 | 无状态 |

---

## 二、常见安全攻击与防御

### 2.1 SQL 注入

```sql
-- 危险
SELECT * FROM user WHERE name = '$name';  
-- 输入: ' OR '1'='1' -- 
-- 结果: SELECT * FROM user WHERE name = '' OR '1'='1' --'

-- 防御：预编译
PreparedStatement ps = conn.prepareStatement("SELECT * FROM user WHERE name = ?");
ps.setString(1, name);
```

### 2.2 XSS（跨站脚本攻击）

```
反射型：恶意脚本在 URL 中，服务端反射回页面
存储型：恶意脚本存到数据库，所有人访问都中招

防御：
1. HTML 转义（< → &lt; > → &gt;）
2. CSP（Content Security Policy）
3. HttpOnly Cookie
4. 输入校验 + 输出编码
```

### 2.3 CSRF（跨站请求伪造）

```
攻击者诱导用户点击链接，利用用户已登录的状态伪造请求

防御：
1. SameSite Cookie（Strict / Lax）
2. CSRF Token
3. Referer/Origin 请求头验证
4. 重要操作二次确认（验证码/密码）
```

### 2.4 DDoS 防御

```
1. CDN / 高防IP
2. Nginx 限流：limit_req + limit_conn
3. IP 黑名单 + 验证码
4. 网关层流量清洗
```

### 2.5 API 安全

```java
// 1. 参数校验
@NotNull @Size(min=1, max=100) String name

// 2. URL 白名单
// 3. 接口签名（防篡改）
String sign = MD5(params + timestamp + secret);
// 4. 接口防重放（timestamp + nonce）

// 5. 敏感数据脱敏
@JsonSerialize(using = PhoneDesensitize.class)
private String phone; // 138****8000

// 6. HTTPS 强制
```

---

## 三、数据安全

### 3.1 密码存储

```java
// ❌ 明文
// ❌ MD5（彩虹表可破）
// ✅ BCrypt（盐值内嵌，慢哈希）
String hash = BCrypt.hashpw(password, BCrypt.gensalt());
boolean match = BCrypt.checkpw(password, hash);

// 或：Argon2（比 BCrypt 更安全）
```

### 3.2 敏感配置加密

```yaml
# Spring Cloud Config + 加密
# 或使用 Vault / KMS 管理密钥
spring:
  datasource:
    password: '{cipher}encrypted-string'
```

### 3.3 日志脱敏

```java
// 日志中不要打印敏感信息（密码、身份证、手机号）
log.info("用户{}登录", username);  // ✅
log.info("密码: {}", password);    // ❌
```

---

## 面试追问集

**Q：如何设计一个权限系统？**

RBAC（Role-Based Access Control）：

```
用户 ←→ 角色 ←→ 权限（资源+操作）
User ─→ Role (admin/editor/viewer) ─→ Permission (user:read, user:write)

表结构：
user → user_role → role → role_permission → permission
```

**Q：如何防止接口被刷？**

1. 限流（单用户/IP 级别的限流）
2. 验证码（人机验证）
3. 前端防抖 + 按钮禁用
4. 请求签名 + timestamp 防重放
5. 行为分析（风控平台）
