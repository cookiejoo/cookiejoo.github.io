---
title: "iframe跨域解决方案"
layout: post
author: 曹德高
tags: [http]
categories: Http
---


## 1.原因现象

应用中如果需要用iframe嵌套某个应用的页面，该应用也是用了灵犀登录，但在iframe中出现无限循环登录。

该现象是在谷歌浏览器中出现，原因是谷歌浏览器限制了嵌入的网页不可设置cookie，导致无法访问。仔细检查，在浏览Set-cookie的响应头出发现提示：

```txt
This Set-Cookie didn't specify a "SameSite" attribute and was defaulted to "SameSite=lax", and was blocked because it came from a cross-site response which was not the response to a top-level navigation. ....
```

原因：谷歌最新版的浏览器默认SameSite=Lax （该设置从2020 年7月14全面展开，具体见： [https://www.chromestatus.com/feature/5088147346030592](https://www.chromestatus.com/feature/5088147346030592)），如果设置SameSite=Lax , 并且嵌入Iframe的地址和iframe外的地址不是SameSite,那么嵌入iframe的地址将无法设置设置cookie。关于什么是SameSite，首先应该理解site，所谓Site就是域名后缀和其前面的第一部分，比如web.dev可以是一个Site，从而 www.web.dev 和 static.web.dev 可以看成相同的Site。具体可以参考SameSite解释 或 [https://tools.ietf.org/html/draft-ietf-httpbis-cookie-same-site-00](https://tools.ietf.org/html/draft-ietf-httpbis-cookie-same-site-00) 。

如此图：由于主页登录过后，iframe的第三方页面无法设置已登录的信息在cookie中导致第三方页面无法获取到登录信息总是显示登录页

![image-20211230102458015](/images/iframe_cors/image-20211230102458015.png)

## 2.SameSite 属性

Cookie 的`SameSite`属性用来限制第三方 Cookie，从而减少安全风险。

它可以设置三个值。

> - Strict
> - Lax
> - None

### 2.1 Strict

`Strict`最为严格，完全禁止第三方 Cookie，跨站点时，任何情况下都不会发送 Cookie。换言之，只有当前网页的 URL 与请求目标一致，才会带上 Cookie。

> ```bash
> Set-Cookie: CookieName=CookieValue; SameSite=Strict; 
> ```

这个规则过于严格，可能造成非常不好的用户体验。比如，当前网页有一个 GitHub 链接，用户点击跳转就不会带有 GitHub 的 Cookie，跳转过去总是未登陆状态。

### 2.2 Lax

`Lax`规则稍稍放宽，大多数情况也是不发送第三方 Cookie，但是导航到目标网址的 Get 请求除外。

> ```markup
> Set-Cookie: CookieName=CookieValue; SameSite=Lax;
> ```

导航到目标网址的 GET 请求，只包括三种情况：链接，预加载请求，GET 表单。

| 请求类型  | 示例                                 | 正常情况    | Lax         |
| :-------- | :----------------------------------- | :---------- | :---------- |
| 链接      | `<a href="..."></a>`                 | 发送 Cookie | 发送 Cookie |
| 预加载    | `<link rel="prerender" href="..."/>` | 发送 Cookie | 发送 Cookie |
| GET 表单  | `<form method="GET" action="...">`   | 发送 Cookie | 发送 Cookie |
| POST 表单 | `<form method="POST" action="...">`  | 发送 Cookie | 不发送      |
| iframe    | `<iframe src="..."></iframe>`        | 发送 Cookie | 不发送      |
| AJAX      | `$.get("...")`                       | 发送 Cookie | 不发送      |
| Image     | `<img src="...">`                    | 发送 Cookie | 不发送      |

### 2.3 None

Chrome 计划将`Lax`变为默认设置。这时，网站可以选择显式关闭`SameSite`属性，将其设为`None`。不过，前提是必须同时设置`Secure`属性（Cookie 只能通过 HTTPS 协议发送），否则无效。

## 3.解决方案

### 3.1解决方案一（不推荐）

低版本的谷歌浏览器(80一下)在地址栏输入：Chrome://flags回车搜索cookies，设置两个值为Disabled，主要是Same-Site这个值是允许第三方设置Cookies属性。（此方法只能让用户去设置浏览器属性并不友好）

![image-20211230102641324](/images/iframe_cors/image-20211230102641324.png)

### 3.2解决方案二（不推荐）

在设置Cookie的响应头中添加 SameSite=None；Secure ，在Cookie响应头设置SameSite=None就可以替换谷歌浏览器的默认配置，而SameSite=None可以允许跨Site设置Cookie。一个完整的cookie响应头可以像这样： Set-Cookie: JSESSIONID=ACD341E8840C03003E4532921C21C035;Path=/begin01; SameSite=None; Secure ，至于怎么加响应头SameSite=None；Secure，可以考虑在nginx里设置，参考：

```bash
# 只支持 proxy 模式下设置，SameSite 不需要可删除(默认不设置则：Lax)，如果想更安全可以把 SameSite 设置为 Strict，我们需要Cookie则是None
proxy_cookie_path / "/; httponly; secure; SameSite=None";

# 示例
server {
    listen 443 ssl http2;
    server_name www.cat73.org;

    ssl_certificate /etc/letsencrypt/live/cat73.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cat73.org/privkey.pem;

    ssl_trusted_certificate /etc/letsencrypt/live/cat73.org/chain.pem;

    add_header X-XSS-Protection "1; mode=block";
    add_header X-Frame-Options SAMEORIGIN;
    add_header Strict-Transport-Security "max-age=15768000";

    location / {
        root /var/www/html;
    }

    location /api {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # 在这里设置
        proxy_cookie_path / "/; httponly; secure; SameSite=None";
    }
}
```

也可以在后台代码里设置，推荐nginx吧。这里注意一点 Secure 属性，这个属性说明设置的该cookie只能在https下传输，所以要求你使用https，不使用行吗，不行，因为SameSite=None 要求必须要和Secure=true一起使用。所以采用这个方法解决，你嵌入iframe的页面必须使用https。

**（我们在本次解决方案中只设置了Cas的登录这边的Ng重启后就有效了，iframe的第三方没有改任何配置。这里为何不推荐使用此方式，是因为如果允许跨域传输cookie会让自身暴露在不安全的环境中，容易遭受CSRF漏洞工具或跨域追踪）**

![20200904161952679](/images/iframe_cors/20200904161952679.png)

### 3.3解决方案三

改造该模式不再使用ifram方式嵌入页面，可用跳转子系统tab页展示第三方应用页面，由于已经全部集成了CAS登录，所以一次登录可多次流量SSO页面，从而避免跨域攻击。

### 3.4解决方案四

页面UI的开发独立到一个应用中如xxx-ui(前后端分离)，请求可以用网关转发即可，此方式缺点在于发布UI要统一版本发布，集中式管理，如果是要用到其他第三方非本公司的系统页面还是无法解决(尽量避免)。优势是后端不受集中式发布影响，可避免嵌入Iframe。

## 4. 跨域攻击CSRF和跨域追踪

### 4.1 跨域攻击

可伪造你的身份去访问受信任的网站，以你名义发送邮件，发消息，盗取你的账号，甚至于购买商品，虚拟货币转账......造成的问题包括：个人隐私泄露以及财产安全。

![WX20211230-114247](/images/iframe_cors/WX20211230-114247.png)

示例：

银行网站A，它以GET/POST请求来完成银行转账的操作，如： http://www.mybank.com/loan.do?toBankId=cdg&money=1000

危险网站B，它里面有一段HTML的代码如下：

<img src= http://www.mybank.com/loan.do?toBankId=hack&money=1000 >

首先，你登录了银行网站A，然后访问危险网站B，噢，这时你会发现你的银行账户少了1000块......

为什么会这样呢？原因是在访问危险网站B的之前，你已经登录了银行网站A，而B中的<img>以GET的方式请求第三方资源（这里的第三方就是指银行网站了，原本这是一个合法的请求，但这里被不法分子利用了），所以你的浏览器会带上你的银行网站A的Cookie发出Get/POST请求，去获取资源“ http://www.mybank.com/loan.do?toBankId=hack&money=1000”，结果银行网站服务器收到请求后，认为这是一个更新资源操作（转账操作），所以就立刻进行转账操作......

### 4.2 跨域追踪

第三方网站引导发出的 Cookie，就称为第三方 Cookie。它除了用于 CSRF 攻击，还可以用于用户追踪。

比如，Facebook 在第三方网站插入一张看不见的图片。

<img src= facebook.com?d=xx style= visibility:hidden >

浏览器加载上面代码时，就会向 Facebook 发出带有 Cookie 的请求，从而 Facebook 就会知道你是谁，访问了什么网站。
