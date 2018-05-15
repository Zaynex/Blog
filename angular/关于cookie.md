关于 Cookie . 不得不说的一些事情。

我记得曾经有一次面试遇到了关于 Cookie 非常简单的问题。然而我竟然答不上来。这使我感到羞愧万分。
只知道之前书里有瞄过几眼，但却很少使用。最近做了一些登录的项目里频繁接触 Cookie。

什么是 Cookie
你可以在很多文章中找到什么是 Cookie。 我觉得最简单的理解就是：服务器在返回请求的时候加了些点心给客户端。
客户端在下一次发送请求的时候会带上这些点心确认这是服务器做的点心。如果是的话，继续做正常流程处理。

## Cookie概貌
首先我们看下 Cookie 长啥样。
![cookie](https://github.com/Zaynex/Blog/blob/master/angular/snapshot/cookie.png?raw=true)

我们看到左侧有个 Cookies。每个站点下面都有一个独立的 Cookie。书上说每个站点的 Cookie 存储空间不超过`4KB`。
图片中心区域就是存储的Cookie。可见 Cookie 有
- name
- value
- domain
- path
- Expires/Max-age
- Size
- HTTP
- Secure
- SameSite
这`9`个属性。其中Size 是浏览器对 Cookie 操作的额外解析长度(name + value)。SameSite 还处于实验阶段，可以要求 Cookie 在跨站点请求时不会被携带发送， 从而阻止跨站请求伪造攻击（csrf）。其余都是比较常用的。
Domain 表示的是哪些主机可以接受 Cookie. path 表示指定主机下哪个子域名可以接受Cookie。如果没有传 path。默认为 /。

Expire/Max-age 表示 Cookie 过期的时间和有效期。注意当设定了 Expire 之后，过期时间是以客户端的时间为准的。

Secure 表明该Cookie 只能通过HTTPS协议的请求发送到客户端。从 Chrome 52 和 Firefox 52 开始，不安全的站点（http:）无法使用Cookie的 Secure 标记。

HttpOnly 表示该 Cookie 无法在客户端被获取，只能发送给客户端。一些服务器的 Session 信息的 Cookie 不想被客户端 JS 脚本调用时，那么应该设置为 HttpOnly。
* 所以 document.cookie 是获取不到全部的 Cookie 的。这是避免脚本被 XSS 攻击。

### 创建Cookie
我们使用 document.cookie 就可以拿到 Cookie ，也可以设置 Cookie 。
![document cookie](https://github.com/Zaynex/Blog/blob/master/angular/snapshot/documetcookie.png?raw=true)

比较有意思的是手动设置 Cookie 时并不会清除站点所有的 Cookie.
```
document.cookie = 'setcookiehh=yes'
```
而是会将新写入的`name=value`累加到 Cookie 里面。不过我们会看到新写入的 Cookie 里面会多了一个空格。这个在后面我们想要获取某个特定的Cookie值时可能需要处理下。


### 轮子：获取指定 Cookie
```
function parseCookieValue(cookieStr, name) {
	name = encodeURIComponent(name)
	for(let cookie of cookieStr.split(';')) {
		const eqIndex = cookie.indexOf('=')
		const [cookieName, cookieValue] = 
			eqIndex == -1 ? [cookie, ''] : [cookie.slice(0, eqIndex), cookie.slice(eqIndex+1)]
			if(cookieName.trim() === name) {
				return decodeURIComponent(name)
			}
	}
	return null
}
```

### 结语
似乎写的这些东西文档里都有，无非是看到了这个 Angular 里面写的这个 Cookie 方法。以后能直接用的方法都可以提出来，自己就可以撸一个库了。
推荐阅读 -[HTTP cookies](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies)
