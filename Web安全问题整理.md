# Web 安全问题整理

在前端的开发中，安全是我们必须要了解的一环。前端是面向用户的，一旦有漏洞，很容易被攻击者发现，了解这些攻击有助于我们如何去规避问题，毕竟安全问题都是需要提前预防，而不能等到真正发生的时候才来解决。
## XSS

> XSS（跨站脚本攻击）全称为 Cross Site Scripting，为了和CSS 文件（Cascading Style Sheets）区分，故称为 XSS。XSS 通过往 Web 页面插入恶意代码，当用户访问该页面时，执行嵌入的恶意代码，以此来达到恶意攻击用户的目的。

之所以会有 XSS 漏洞，本质是浏览器无法识别那些脚本资源是合法的（所以后面出了个 CSP 内容安全策略规范，可以识别那些资源是合法的）。

只要有 XSS 漏洞，那么这个网站对用户来说已经十分不安全了，用户的 cookie 等浏览器的用户本地数据对攻击者来说已经不是什么秘密。

XSS 目前是前端遇到的最多的安全问题之一，如果你没这个意识，那么就很有可能出现 XSS 漏洞。

### 简单的 XSS 漏洞例子

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
  </head>
  <body>
    <div id="app"></div>
    <script>
      document.querySelector('#app').innerHTML = `当前访问的网址是${
        location.href
      }`
    </script>
  </body>
</html>
```

在 IE8 以下的老版本浏览器，使用 `http://examle.com/?type=<script>alert("xss")</script>` 访问就会弹出 `xss` 字符串。

不过现在的主流浏览器都做了一些特殊字符串的转换上面的链接会转换为 `[http://172.22.88.37:5000/test?type=%3Cscript%3Ealert(%22xss%22)%3C/script%3E](http://172.22.88.37:5000/test?type=alert("xss"))`。而且即使使用如下代码在现代浏览器也是没效果的，浏览器本身做了防御，不允许字符串形式创建 script 标签：

```js
document.body.innerHtml = "<script>alert('xss')</script>"
```

### 什么情况下可能会出现 XSS 漏洞呢？

XSS 的本质在于浏览器无法识别脚本是当前网站的脚本，只要是脚本就会去运行，那么非本页面的脚本如果可以注入，那么就可以运行。

- 反射型 XSS

  url 的参数直接展示到页面上。

- 存储型 XSS

  后端接口用户数据不做处理，直接展示到页面上。

- DOM 型 XSS

  - 用户的跳转 url

    在标签的 href、src 等属性中，包含 `javascript:` 等可执行代码，如下面的的 url 注入就会弹出 XSS 字符串，现在的浏览量也没办法避免这个 XSS

    ```html
    <a id="dd" href="javascript:alert(&#x27;XSS&#x27;)">跳转...</a>
    <script>
    document
        .querySelector('#dd')
      .setAttribute('href', location.href.split('type=')[1])
    </script>
    ```
  
    访问 [http://example.com?type=javascript:alert(%27XSS%27)](http://172.22.88.37:5000/test?type=javascript:alert('XSS'))，然后点击跳转就会弹出 XSS。
  
  - 在 onload、onerror、onclick 等中，注入不受控制代码
  
    DOM 中的内联事件监听器，如 `location`、`onclick`、`onerror`、`onload`、`onmouseover` 等，还有`JavaScript` 的 `eval()`、`setTimeout()`、`setInterval()` 等，都能把字符串作为代码运行。如果不可信的数据拼接到字符串中传递给这些 API，很容易产生安全隐患，请务必避免。
    
    这种情况基本不会出现，除非开发者那么无聊。
    
    如下点击 test 也是会弹出 0 的。
    
    ```html
    <div onclick="alert(0)">
      test
    </div>
    ```

### 如何避免 XSS 漏洞

**这里都是在前后端分离的情况下的防御，如果是直出的 html 代码（服务端渲染，包括传统服务端渲染和 node 同构渲染），那么后端输出的非页面本身的 html 代码就要转义和过滤了，后端直出 html 代码比前端代码插入节点危险多了。**

这里主要讲的是前后端分离的前端防范措施。

#### 代码层的措施

一下解决方案都是需要配合使用的，不是单个使用就可以防范的了。

- 比较保障的办法就是前端展示数据的**转义**

  虽然现在主流浏览器都做了 script 标签的注入静默处理（注入的 script 代码是不会运行的），但是其他的标签是会渲染的，也会导致界面错乱、iframe 内嵌等等问题。所以还是要进行 html 标签字符转义，同时也兼容老的浏览器。

- 用户输入或者存储的长度控制

  这可以部分避免复杂代码的 XSS 漏洞，但是 XSS 漏洞如果存在，还是有问题的。

- 关键字过滤

  用户自定义的跳转 url，需要进行 `javascript:` 关键字过滤

- 避免向 `onclick` 、`eval` 等 API传递用户数据

  如下面的代码，加上 `alert(0)` 使用户的输入信息，那么你就是傻傻直接开了个们给攻击者，当然这个基本没人会犯这个低级错误。

如果是使用如 React、Vue 这些框架，框架本身就进行了相关防范，直接使用框架自带的渲染，不要使用 `v-html`/`dangerouslySetInnerHTML` 等这类功能进行 html 渲染，那么基本都不用担心出现 XSS 漏洞的问题。

如果真的需要进行转义，本人建议不要自己写转义的代码，使用已有公认的转义库，这样会避免有遗漏。

#### 其他的措施

##### 内容安全策略   ([CSP](https://developer.mozilla.org/en-US/docs/Glossary/CSP)) 

内容安全策略   ([CSP](https://developer.mozilla.org/en-US/docs/Glossary/CSP)) 是一个额外的安全层，用于检测并削弱某些特定类型的攻击，包括跨站脚本 ([XSS](https://developer.mozilla.org/en-US/docs/Glossary/XSS)) 和数据注入攻击等。

CSP 的主要目标是**减少和报告** XSS 攻击，有些情况还是需要代码层的处理。

详细的用法看[内容安全策略( CSP )](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP)的介绍。

## CSRF

> CSRF（Cross-site request forgery）跨站请求伪造：攻击者诱导受害者进入第三方网站，在第三方网站中，向被攻击网站发送跨站请求。利用受害者在被攻击网站已经获取的注册凭证，绕过后台的用户验证，达到冒充用户对被攻击的网站执行某项操作的目的。

CSRF 的本质在于**伪造**，CSRF 无法像 XSS 一样获取用户的数据，但是却是利用用户已登录的状态进行伪造攻击。要了解这一点，你必须知道 cookie 的特性，CSRF 就是利用同域名 cookie 会自动回传给服务端的特性，进行伪造攻击（真实用户没有进行操作，攻击者伪造请求进行相关操作）。

### 简单例子

> 小明经常在 **A** 银行网站进行转账，某一天他**刚刚**转完账，特然电脑弹出了个广告，他比较好奇，就点进去了 **B** 网站，然后就并没有什么值得看的，然后就退出了。然后过了几分钟，他的手机就收到信息：“您的转账已成功到账 100000 元”。然后他蒙了，自己没有转这笔账啊，然后赶去去看账户余额，发现钱真的不见了。

这里的例子只是为了说明 CSRF 是怎么进行攻击的（现实中当然没这么不安全的，还需要手机验证等等），我们先来分析下，攻击者是如何进行攻击的。

首先需要用户在 **A** 网站进行了转账而且登录态没失效（session cookie），然后通过广告等形式诱导用户进入 **B** 网站。**B**网站的攻击代码如下：

```html
<form
  method="POST"
  action="https://www.b.com/transfer/accounts"
  enctype="multipart/form-data"
>
  <input type="hidden" name="count" value="100000" />
  <input type="hidden" name="card-no" value="6217211107001880725" />
</form>
<script>
  document.forms[0].submit()
</script>
```

`submit` 后就达到了伪造用户进行转账。

### 如何防范？

首先我们知道伪造的根本在于 **cookie** 登录态，只要不使用 cookie 登录态，采用 **Http 请求头 token 模式**，就可以避免这个问题，一般这种只能用于前后端分离模式。

如果是后端直出渲染，不使用 Http 请求头 token 模式，我们就需要进行相关防范了。

- 来源检测

  以前后端直出渲染的网址都是同域名的（至少二级域名是要一样），我们可以检测 Http 请求头 `Origin` 和 `Referer` 是否匹配当前域名。

  但是这种方法并非万无一失，Referer 的值是由浏览器提供的，虽然 HTTP 协议上有明确的要求，但是每个浏览器对于 Referer 的具体实现可能有差别，并不能保证浏览器自身没有安全漏洞。使用验证 Referer 值的方法，就是把安全性都依赖于第三方（即浏览器）来保障，从理论上来讲，这样并不是很安全。在部分情况下，攻击者可以隐藏，甚至修改自己请求的来源。

- csrf token

  CSRF 攻击者既然只能伪造，不能窃取用户信息，那么我们可以在页面载入的时候就返回一个 token，用户提交信息的时候需要回传 token，那么攻击者是无法获取到 token，就无法回传了。

  如上面的转账伪造攻击例子，攻击者不知道 token 是多少，提交就是通不过的。

  ```html
  <form
    method="POST"
    action="https://www.b.com/transfer/accounts"
    enctype="multipart/form-data"
  >
    <input type="hidden" name="count" value="100000" />
    <input type="hidden" name="card-no" value="6217211107001880725" />
    <!--攻击者不知道 token 是多少，提交就是通不过的-->
    <!--<input type="hidden" name="csrf-token" value="" />-->
  </form>
  <script>
    document.forms[0].submit()
  </script>
  ```

- 双重 cookie

  在会话中存储 csrf token 比较繁琐，而且不能在通用的拦截上统一处理所有的接口。

  利用 csrf 攻击不能获取到用户 cookie 的特点和 cookie 同域名自动回传的特点，我们可以要求 Ajax 和表单请求携带一个 cookie 中的值，步骤如下：

  - 进入页面的时候前端随机生成 cookie 的值（crsftcookie 属于当前域名的）
  - 用户请求的时候，所用请求 url 都要上面都要附带这个 cookie ，如 `http://www.test.com/?crsftcookie=xxxxx`。

  - 后端接口验证 cookie 中的字段与 url 参数中的字段 crsftcookie 是否一致，不一致则拒绝。

- 使用 cookie SameSite 特点

  ```
  Set-Cookie: <cookie-name>=<cookie-value>; SameSite=Strict
  ```

  允许服务器设定一则 cookie 不随着跨域请求（此跨域跟前后端分离的 cors 跨域不同概念）一起发送，这样可以在一定程度上防范跨站请求伪造攻击。

  但是这个有兼容性，IE 都不支持，所以这个目前还是应用不起来，因为还是有人再用 IE。

### 总结

csrf 是伪造攻击，需要用户登录，而且用户进入了其他第三方网站（所谓的钓鱼网站）才有可能受到攻击。我们只要有手段识别到请求是否属于用户合法请求就可以防范。

## 劫持

劫持比较少出现，前端也没任何手段进行防止。常见有两种劫持方式，分别是 DNS 劫持、HTTP 劫持。

### DNS 劫持

这种劫持会把你重新定位到其它网站，我们所熟悉的钓鱼网站就是这个原理。但是因为它的违法性，现在被严厉的监管起来，已经很少见。

### HTTP 劫持

HTTP 劫持比较常见，特别是没使用正规带宽的网络，中间的运营商可以，往网页上在里面植入广告或者是不是跳转到另外一个网页上。

一起 HTTP1.1 版本时代，劫持在一些中间运营商中，还是比较容易出现劫持的问题。前端没办法避免这个问题，就通过一些手段检测、移除等手段做一些简单处理。劫持手段层出不穷，前端也不可能随时针对性的处理。

现在有 HTTPS 了，劫持已经不是问题，已经从根源上解决了。

那么 HTTPS 也能被劫持吗？

通过伪造证书，可以劫持，但是用户浏览器会警告此网站非安全，如果用户跳过这个警告继续访问，那么就有可能被劫持。完全符合 HTTPS 规则的劫持手段，本人还没见过，也还没弄明白这种厉害的手段。

还有人就说了，我可以让用户回落到HTTP协议啊，中间人用 HTTPS 跟服务器通信，然后用HTTP跟客户端通信，这种方式的前提是服务器没配置 HTTP 强制跳转 HTTPS，才能做到，只要利用 [HTTP Strict Transport Security](https://developer.mozilla.org/zh-CN/docs/Security/HTTP_Strict_Transport_Security) 就可以解决这个问题（后端页面响应头返回 Strict-Transport-Security）。

**示例**

现在和未来的所有子域名会自动使用 HTTPS 连接长达一年。同时阻止了只能通过 HTTP 访问的内容。

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

## iframe 安全问题

iframe 安全问题比较少，不过也还是可能出现安全问题的。

### iframe 内嵌第三方网页

有些时候我们的前端页面需要用到第三方提供的页面组件，通常会以 iframe 的方式引入。典型的例子是使用iframe 在页面上添加第三方提供的广告、天气预报、社交分享插件等等。

iframe 有跨域的防御，第三方无法访问本页面，但是可以让页面**跳转**到其他页面，虽然没泄漏什么用户信息，但是影响体验。

那怎么解决呢？这个也挺好处理的，iframe 有 **sandbox** 属性:

```
allow-forms
allow-modals
allow-orientation-lock
allow-pointer-lock
allow-popups
allow-popups-to-escape-sandbox
allow-same-origin
allow-scripts
allow-top-navigation
```

如果不希望 iframe 跳转到其他页面，不要加上 `allow-top-navigation` 即可。

```html
<iframe sandbox="allow-forms allow-scripts">
</iframe>
```

### 被第三方网站 iframe 内嵌

当然这种情况也会很少出现（被劫持或者用户被钓鱼了），不同主域名，第三方网站也没办法获取 iframe 加载的网站，但是也可以打点小广告之类的。

防止自己的网站被其他网站的 `iframe` 引用，需要后端配合，在 http 响应头中返回 `X-Frame-Options: DENY` 即可。

`X-Frame-Options` 有三个可能值: `DENY`， `SAMEORIGIN`， `ALLOW-FROM xxx(填写url)`。

### SQL 注入攻击



## DoS 攻击



## 参考文章

- [前端安全系列（一）：如何防止XSS攻击？](https://juejin.im/post/5bad9140e51d450e935c6d64#heading-13)

- [内容安全策略( CSP )](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP)

- [HTTP Strict Transport Security](https://developer.mozilla.org/zh-CN/docs/Security/HTTP_Strict_Transport_Security)

  