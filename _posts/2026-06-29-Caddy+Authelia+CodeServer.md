---
title: 【域名+Caddy+Authelia+Code-Server】为简单Web程序添加安全防护
# author: Red2Yang
date: 2026-06-29 10:00:00 +0800
categories: [网络, 公网]    # 最多两个层级
tags: [Caddy,Authelia,Code-Server,网络安全]      # 小写，多个用空格或逗号
---

## 引言

如果你尝试使用过`Jupyter Lab`或者`Code-Server`这些网页端开发工具，不难发现它们两个的安全防护可以说是“难兄难弟”，都只要直到密码或者用户名+密码就能使用了。`Code-Server`的安全问题更加严重，**登录后使用者可以访问所有文件目录**。因此，它们是远远不能达到部署在公网上的标准的。当然，部署在公网完全不是为了他人使用，因为本身它们就没有设计过多人协同作业的功能，但是，我们仍然要防范各式的网络攻击和密码爆破，避免服务器瘫痪或被他人掌握。

通常论坛、网站会选择Cloudflare等网络巨头提供域名防护和认证功能，所以这里我们不考虑网络攻击和人机验证。只针对一件事：如何提高登录复杂度，避免密码被他人知道或破解后直接登录进入。常见的方法是双因素认证方法，以密码+验证码/通行密钥的形式，也可以将密码设置得非常简单，这样就是单验证码因素方法。攻击者登入后，无法得知认证软件的动态验证码，也没有通行密钥（除非你的设备被盗），从而无法破解。即使攻击者尝试修改双因素方式，比如删除动态验证码验证，但这本身需要认证内部系统的准入权或邮箱登录权，成功形成防御闭环。

建议使用HTTPS协议，杜绝慢速攻击的可能。

**本文代码使用了人工智能生成技术，请注意辨别**

## Authelia

Authelia是一款开源的Web认证服务软件，它可以和反代软件配合，从而可以上到域名，下到端口进行认证控制。认证软件不是加密软件，因此开源并不会影响其防护逻辑和性能。[这里访问仓库链接](https://github.com/authelia/authelia)。

### 服务端部署

通常我们会选择docker容器进行部署。这里直接给出最便捷的docker-compose部署方法：

```docker-compose.yml
version: '3.8'
services:
  authelia:
    image: authelia/authelia:latest # 镜像名
    container_name: authelia
    volumes:
      - ./config:/config # 配置文件
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=Asia/Shanghai
    restart: unless-stopped
    ports:
      - 8001:8001 
```

输入`docker-compose up -d`即可自动拉取镜像、生成容器并运行，之后该容器会在开机docker启动后自动启动。

### 配置示例文件
接下来是配置文件，在`./config`位置创建`configuration.yml`，然后填写配置。注意，不同版本差别很大，我的版本是**`v4.39.20`**。具体请参照项目仓库进行查询。

```configuration.yml
server:
  address: 'tcp://0.0.0.0:8001/' # 监听地址和端口

log:
  level: debug

authentication_backend:
  file:
    path: /config/users_database.yml # 用户名单，见后文

storage:
  encryption_key: '57d3643dc2c0cf2e70da0dd37e366fbd96b721a1cec8473cda8103feecc2c84b' # openssl rand -hex 64 生成字符串，防止数据库注入破解
  local:
    path: /config/db.sqlite3

totp:
  issuer: auth.1.com # 显示名称，可以任意

session:
  secret: '57d3643dc2c0cf2e70da0dd37e366fbd96b721a1cec8473cda8103feecc2c84b' 
  expiration: 5h # 认证强制过期时间
  inactivity: 20m # 认证空闲过期时间
  cookies:
    - name: 'authelia_session'
      domain: '1.com'
      authelia_url: 'https://auth.1.com' # 该认证服务的域名，不支持子路由，支持子域名
      default_redirection_url: 'https://homepage.1.com/' # 默认认证通过后跳转的页面

identity_validation:
  reset_password:
    jwt_secret: '57d3643dc2c0cf2e70da0dd37e366fbd96b721a1cec8473cda8103feecc2c84b'

access_control:
  default_policy: deny
  rules:
    - domain: 'code-server.1.com' # 控制你的域名，当Caddy发送检查认证请求时，必须通过双因素认证才可访问该域名，否则跳转到登录页面
      policy: two_factor

notifier:
  filesystem:
    filename: /config/notification.txt # 使用系统内验证，添加双因素方法时需查看该处
```

### 用户数据

遗憾的是，哈希密码只能在有docker镜像的机器上从明文生成。之后可以通过后台进行修改。

哈希密码命令如下：

```
docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'password'
```

示例文件，在`./config`位置创建`users_database.yml`

```users_database.yml
# yaml-language-server: $schema=https://www.authelia.com/schemas/latest/json-schema/user-database.json
users:
  john:
    disabled: false
    displayname: 'John Doe'
    password: '$argon2id$v=19$m=65536,t=3,p=2$BpLnfgDsc2WD8F2q$o/vzA4myCqZZ36bUGsDY//8mKUYNZZaR0t4MFFSs+iM'
    email: 'john.doe@authelia.com'
    groups:
      - 'admins'
      - 'dev'
  harry:
    disabled: false
    displayname: 'Harry Potter'
    password: '$argon2id$v=19$m=65536,t=3,p=2$BpLnfgDsc2WD8F2q$o/vzA4myCqZZ36bUGsDY//8mKUYNZZaR0t4MFFSs+iM'
    email: 'harry.potter@authelia.com'
    groups: []
  james:
    disabled: false
    displayname: 'James Dean'
    password: '$argon2id$v=19$m=65536,t=3,p=2$BpLnfgDsc2WD8F2q$o/vzA4myCqZZ36bUGsDY//8mKUYNZZaR0t4MFFSs+iM'
    email: 'james.dean@authelia.com'
    groups: []
  bob:
    disabled: false
    displayname: 'Bob Dylan'
    password: '$argon2id$v=19$m=65536,t=3,p=2$BpLnfgDsc2WD8F2q$o/vzA4myCqZZ36bUGsDY//8mKUYNZZaR0t4MFFSs+iM'
    email: 'bob.dylan@authelia.com'
    groups:
      - 'dev'
    given_name: 'Robert'
    family_name: 'Zimmerman'
    middle_name: 'Allen'
    nickname: 'Bob'
    profile: 'https://en.wikipedia.org/wiki/Bob_Dylan'
    picture: 'https://kelvinokaforart.com/wp-content/uploads/2023/01/Bob-Dylan.jpg'
    website: 'https://www.bobdylan.com/'
    gender: 'male'
    birthdate: '1941-05-24'
    zoneinfo: 'America/Chicago'
    locale: 'en-US'
    phone_number: '+1 (425) 555-1212'
    phone_extension: '1000'
    address:
      street_address: '2-3 Kitanomarukoen'
      locality: 'Chiyoda City'
      region: 'Tokyo'
      postal_code: '102-8321'
      country: 'Japan'
    extra:
      example: 'value'
```

## Caddy

Caddy是一个和nginx功能相同的软件。详细内容可以参考Caddy中文教程。

```Caddyfile
# 认证
auth.1.com:80 {
    reverse_proxy localhost:8001 {
        header_up X-Forwarded-Proto https
    }
}
code.1.com:80 {
    @websockets {
        header Connection *Upgrade*
        header Upgrade websocket
    }

    # 🔑 新增：如果是 WebSocket 请求，直接代理给 code-server，跳过 Authelia 验证！
    # 因为能发起 WebSocket 的前提是页面已经加载（说明已经通过了 2FA 验证）
    handle @websockets {
        reverse_proxy localhost:8012 { # 要认证的服务，这里是code-server
            # 必须保留，解决终端输出延迟
            flush_interval -1
        }
    }
    reverse_proxy localhost:8001 {
        # 1. 强制覆盖协议头为 https (必须在 rewrite 之前，这样 Authelia 才能收到正确的 Proto)
        header_up X-Forwarded-Proto https
        
        # 2. 始终使用 GET，避免消耗传入请求的请求体
        method GET
        
        # 3. 将 URI 改为认证网关的验证端点
        # ⚠️ rd 参数必须是 https，这样未登录时才能正确重定向回 Authelia
        rewrite /api/verify?rd=https://auth.1.com/
        
        # 4. 转发原始方法和 URI (补充 Authelia 需要的上下文)
        header_up X-Forwarded-Method {method}
        header_up X-Forwarded-Uri {uri}

        # 5. 认证成功 (2xx) 时的处理逻辑
        @good status 2xx
        handle_response @good {
            # 将 Authelia 返回的身份头注入到即将发给 code-server 的原始请求中
            request_header Remote-User {rp.header.Remote-User}
            request_header Remote-Email {rp.header.Remote-Email}
            request_header Remote-Groups {rp.header.Remote-Groups}
            request_header Remote-Name {rp.header.Remote-Name}
        }
    }

    # 只有上面的 handle_response 放行后，才会执行这里的反向代理
    reverse_proxy localhost:8012 { # 要认证的服务
        header_up X-Forwarded-Proto https
        # ⚠️ 针对 code-server 的关键优化：解决 WebSocket 终端输出延迟
        flush_interval -1
        
        header_up X-Forwarded-Proto https
        # 告诉 code-server 外部访问的域名是什么
        header_up X-Forwarded-Host {host}
        # 传递原始 Host
        header_up Host {host}
        # 传递真实客户端 IP
        header_up X-Real-IP {remote_host}
        # 传递完整的转发链路信息
        header_up X-Forwarded-For {remote_host}
        
        # 支持 WebSocket 升级
        header_up Connection {>Connection}
        header_up Upgrade {>Upgrade}
    }
}
```

同时运行caddy和authelia，就可以正常验证了。
