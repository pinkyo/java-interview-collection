# Nginx

---

## 一、正向代理 vs 反向代理

```
正向代理：
  客户端 ──► 代理服务器 ──► 目标服务器
  （代理客户端，隐藏客户端 IP）

反向代理：
  客户端 ──► Nginx ──► 后端服务器
  （代理服务端，隐藏服务端 IP）
```

---

## 二、常用功能

### 2.1 反向代理与负载均衡

```nginx
upstream backend {
    server 192.168.1.10:8080 weight=5;
    server 192.168.1.11:8080 weight=3;
    server 192.168.1.12:8080 backup;  # 备用节点
    server 192.168.1.13:8080 down;    # 下线
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### 2.2 负载均衡算法

| 算法 | 指令 | 说明 |
|------|------|------|
| **轮询** | 默认 | 依次分配 |
| **加权轮询** | weight=N | 按权重分配 |
| **IP Hash** | ip_hash | 同一 IP 固定到一台服务器（session 保持） |
| **最少连接** | least_conn | 分配给连接数最少的服务器 |
| **URL Hash** | hash $request_uri | 按 URL 分配（缓存友好） |
| **Fair** | fair（第三方） | 按响应时间分配 |

### 2.3 限流

```nginx
# 限制请求速率（令牌桶）
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=mylimit burst=20 nodelay;
        proxy_pass http://backend;
    }
}

# 限制并发连接数
limit_conn_zone $binary_remote_addr zone=addr:10m;
limit_conn addr 10;
```

---

## 三、Nginx 高并发模型

```
                 ┌────────────────┐
                 │  Master Process│  (处理配置/管理 Worker)
                 └───────┬────────┘
                         │
       ┌─────────┬───────┼───────┬─────────┐
       ▼         ▼       ▼       ▼         ▼
   ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
   │Worker │ │Worker │ │Worker │ │Worker │ │Worker │
   │   1   │ │   2   │ │   3   │ │   4   │ │   n   │
   └───────┘ └───────┘ └───────┘ └───────┘ └───────┘
    (每个 Worker 单线程，epoll 多路复用，处理数万连接)
```

**追问：Nginx 为什么能支持高并发？**

1. **Master-Worker 多进程模型**：Worker 数量 = CPU 核数，充分利用多核
2. **epoll I/O 多路复用**：单线程处理数万连接
3. **异步非阻塞**：Worker 不阻塞等待，快速切换处理
4. **零拷贝**：sendfile 直接在内核态传输文件

---

## 四、常见配置

### 4.1 静态资源

```nginx
location /static/ {
    root /data/www;
    expires 7d;
    add_header Cache-Control "public, immutable";
}
```

### 4.2 动静分离

```nginx
# 动态请求 → 后端
location /api/ {
    proxy_pass http://backend;
}

# 静态文件 → Nginx
location ~* \.(html|css|js|png|jpg|gif)$ {
    root /data/www;
}
```

### 4.3 HTTPS 配置

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
}
```

### 4.4 跨域配置

```nginx
location /api/ {
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'Authorization, Content-Type';

    if ($request_method = 'OPTIONS') {
        return 204;
    }
}
```

---

## 五、Nginx + Lua（OpenResty）

OpenResty 是 Nginx + LuaJIT 的扩展平台，可在 Nginx 中编写 Lua 脚本实现高扩展功能。

```lua
-- 限流示例
local limit_req = require "resty.limit.req"
local lim, err = limit_req.new("my_limit_req_store", 10, 5)
local delay, err = lim:incoming(key, true)
if not delay then
    ngx.status = 503
    ngx.exit(503)
end
```

适用场景：动态限流、WAF、API 网关、AB 测试。

---

## 面试追问集

**Q：Nginx 如何处理一个请求？**

1. Master 进程 fork 出 Worker 进程
2. Worker 进程通过 epoll 监听事件
3. 请求到达 → accept → 解析 HTTP → 匹配 server/location → 代理/静态处理 → 返回响应

**Q：Nginx 的 `rewrite` 和 `proxy_pass` 的区别？**

- `rewrite`：URL 重写，改变请求 URI，仍在 Nginx 内部处理
- `proxy_pass`：反向代理，将请求转发到后端服务器

**Q：负载均衡时如何保持 Session？**

1. `ip_hash`：同一 IP 始终路由到同一台服务器
2. `sticky cookie`（第三方模块）：Nginx 注入 Cookie
3. 应用层：Redis 共享 Session（推荐）
