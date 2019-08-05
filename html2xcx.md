# 非结构化的富文本数据在小程序中的实践。
结构化的数据比较容易处理，按内容渲染即可。这篇着重讲非结构化的数据在小程序的显示。  
主要有三部：  

- 将html字符串转换为AST 语法树
- 将AST 语法树的结构化数据渲染wxml
- 处理图片和预览大图

## 第一步：转换为 AST 语法树

后端返回的是一个字符串，首先我们需要转换成结构化的数据对象。字符串转换成对象，有且只有一种了，使用正则表达式来查找字符串当中 Html 标签等信息了。首先我们来看三个正则表达式：

- 标签的起始标识
```js
const startTag = /^<([-A-Za-z0-9_]+)((?:\s+[a-zA-Z_:][-a-zA-Z0-9_:.]*(?:\s*=\s*(?:(?:"[^>]*")|(?:'[^>]*')|[^>\s]+))?)*)\s*(\/?)>/;
```
标签的起始正则：  
已<开头紧跟着横线、字母、数字或下划线，然后跟着 1 到多个空格，然后跟着 1 个字母、下划线或者冒号，然后跟着多个横线、字母、数字或下划线，再跟着 0 或者多个空格，再跟着=，0 或者多个空格，再跟着（0 或 1 个“号，多个不是>的字符，然后跟着 0 或 1 个”好）或者（跟着一个‘，多个不是>的字符，然后一个’）或者(不是>和空格的多个字符),然后跟着 0 或 1 个/，然后以>结尾。  
例子：<div id="abcde"\> 或者 <img src="xxx" \/>

- 标签的结尾

```js
const endTag = /^<\/([-A-Za-z0-9_]+)[^>]*>/;
```

标签的结尾正则：已<\/开头跟着多个横线、字母、数字或下划线，然后多个不是>的字符，然后跟着>结尾  
例子：</div>

- 标签的属性

```js
const attr = /([a-zA-Z_:][-a-zA-Z0-9_:.]*)(?:\s*=\s*(?:(?:"((?:\\.|[^"])*)")|(?:'((?:\\.|[^'])*)')|([^>\s]+)))?/g;
```

标签的属性的正则：已字母下滑线开头，跟着横线、字母、数字或下划线冒号.的多个字符，然后跟着多个空格，=号，然后跟着多个空格，然后是（“开头，跟着不含”的多个字符，然后跟着”）或者（‘开头，跟着不含’的多个字符，然后跟着‘）或者（不含>和空格的多个字符）。

### HTML parser：找到所有的节点起始和结束标签

定义变量 stack 栈存储未截止的标签，提供 last 方法，pop 最近的一个标签。

```js
function HTMLParser(html, handler) {
  const stack = [];
  stack.last = function() {
    return this[this.length - 1];
  };
  while(html&flag){
    ……
  }
  parseEndTag();// 统一未处理的所有的起始标签。
}
```

如果内容已”<\!--“开头的，则认为是注释，立即查找注释结束标识”-->“，如果找到了，则将 html 的注释截掉，如果 parse 需要处理 comment，则回调 handler.comment 处理。

```js
if (html.indexOf("<!--") == 0) {
  index = html.indexOf("-->");
  if (index >= 0) {
    if (handler.comment) {
      handler.comment(html.substring(4, index));
    }
    html = html.substring(index + 3);
    chars = false;
  }

  // end tag
}
```

如果检测到”</“，则认为是某个标签的截止符号，匹配 endTag 的正则表达式，如果匹配上了，则将 html 的此 endTag 给去除，留下准备处理后面的字符串。看 endTag 的正则，match[0]为匹配的整个 endTag，我们将匹配的 endTag,来处理，parseEndTag.

```js
  else if (html.indexOf('</') == 0) {
      match = html.match(endTag);
      if (match) {
          html = html.substring(match[0].length);
          match[0].replace(endTag, parseEndTag);
          chars = false;
      }
  }
```

endTag 的正则的结果的 match[0]是整个 endTag，match[1]则为标签符号（不明白可以复习 js 正则的 match）。如果 tagName 为空，pos 置位 0，否则即找到了一个结尾标识 tagName，因此循环 stack，从后往前找到和 tagName 匹配的起始标签（html 是标签先打开先闭合，后打开后闭合），如果找到了，我们认为找到了结束标签对应的起始标签，并且忽略这个起始标签之后的起始标签。比如： 、、<div\><span\></div\>，stach 会推入两个标签，div 和 span，但是找到的第一个结束标签是</div\>,则会匹配第一个起始标签<div\>，那么中间的 span 就认为是一个作废的标签，也匹配不到截止标签了。

```js
function parseEndTag(tag, tagName) {
  let pos = 0;
  if (!tagName) {
    pos = 0;
  } else {
    for (pos = stack.length - 1; pos >= 0; pos--) {
      if (stack[pos] == tagName) {
        break;
      }
    }
  }
  if (pos >= 0) {
    for (let i = stack.length - 1; i >= pos; i--) {
      if (handler.end) {
        handler.end(stack[i]);
      }
    }
    stack.length = pos;
  }
}
```

如果检测到已"<"开头的，认为是起始标签(前面已经判断完<\!--和</,如果 html 有自己需要特殊判断的标签，比如组件<\component 的处理则需要放在前面判断)。match 起始标签 startTag（起始标签有点特殊，需要看起始标签上的 class，attribute，style 等）。然后同理，将 html 内容后移，去掉已经匹配的 startTag，然后处理起始标签，parseStartTag

```js
else if (html.indexOf('<') == 0) {
    match = html.match(startTag);
    if (match) {
        html = html.substring(match[0].length);
        match[0].replace(startTag, parseStartTag);
        chars = false;
    }
}
```

处理起始标签稍微复杂一点，正文里面有详细的解析了.[1]：<span\><div\>111</div\>,这种我们认为 span 是一个废弃的起始标签，自动截止即可。[2]如果是 dd 等自闭和标签，如<dd\><dd\>，我们认为第二个 dd 标签就是截止标签。

```js
function parseStartTag(tag, tagName, rest) {
  tagName = tagName.toLowerCase();
  if (block[tagName]) {
    //如果这个标签是块状标签，比如div，h1等，
    while (stack.last() && inline[stack.last()]) {
      //当前推入的stack有起始标签，且最近推入的起始标签是行内标签（比如span,img ）
      parseEndTag("", stack.last()); //则将推入的最近的行内起始标签当做截止标签处理，且循环处理完。[1]
    }
  }
  if (closeSelf[tagName] && stack.last() == tagName) {
    //如果当前标签是自闭和标签（比如li等没有嵌套的标签），那我们也可以看最近推入的标签是不是和当前标签一样，如果是，则把当前标签当做截止标签处理。[2]
    parseEndTag("", tagName);
  }
  var unary = empty[tagName]; //如果当前标签是empty元素(比如br)，则标签截止
  if (!unary && tag.match(/\/>$/)) {
    //不是empty元素，但是标签已/>结尾的，则标签也截止
    unary = true;
  }
  if (!unary) {
    //如果标签未截止，则推入该起始标签
    stack.push(tagName);
  }
  if (handler.start) {
    const ds = {};
    let dataItem = "";
    // 起始标签上的东东，获取data-属性，将data-type组件剥离出来，这是我们的自定义组件。
    rest = rest.replace(/data-(\w+)=([^\s]+)/g, function(match, name, value) {
      if (value) value = value.replace(/^['|"]/, "").replace(/['|"]$/, "");
      if (name == "type") {
        dataItem = value;
      } else {
        ds[name] = value;
      }
      return "";
    });
    const attrs = [];
    // 匹配标签上的attr属性，使用属性正则。
    rest.replace(attr, function(match, name) {
      const value = arguments[2]
        ? arguments[2] //属性是a="b"
        : arguments[3]
        ? arguments[3] // 属性是a='b'
        : arguments[4]
        ? arguments[4] // 属性是a=b
        : fillAttrs[name] // 标签是否是checked,disabled等，如果是的，直接使用name 当value
        ? name
        : "";
      attrs.push({
        name,
        value,
        escaped: value.replace(/(^|[^\\])"/g, '$1\\"') // "
      });
    });
    // 标签名称，属性，是否是截止标签，dataset值和组件返回给start
    handler.start(tagName, attrs, unary, ds, dataItem);
  }
}
```

如果上面都没有匹配到，那就就认为是字符，即标签内的文本内容。然后寻找下一个"<"的位置，然后在下一个"<"之前的都是文本内容，html 然后往前一到下一个标签起始位置。进行下一轮循环。

```js
if (chars) {
  index = html.indexOf("<");
  const text = index < 0 ? html : html.substring(0, index);
  html = index < 0 ? "" : html.substring(index);
  if (handler.chars) {
    handler.chars(text);
  }
}
```

一轮结束，如果 html 和上一次保存的 last 是一致，即 html 内容没有减少，说明出问题了。结束循环。

```js
if (html == last) {
  console.log("Parse Error: " + html);
  flag = false;
}
last = html;
```

### HTML JSON：经过 parse 之后的回调，组装 AST

记录富文本中的图片，用户点击图片，需要大图查看。根节点是 root

```js
function html2json(html, defaultHeight) {
  var wxImageArray = [];
  const bufArray = [];
  const results = {
    node: "root",
    child: []
  };
  HTMLParser(html, {
    start,
    end,
    chars
  });
}
```
html parse 遇到起始标签，开始组装起始 node。定义一个初始的 node，用 bufArray 存储未截止的起始标签。
```js
function start(tag, attrs, unary, dataset, dataItem) {
  const node = {
    node: "element",
    tag,
    attr: {
      style: ""
    },
    data: dataset,
    type: dataItem
  };
  if (attrs.length !== 0) {
    // parseStartTag里面正则匹配到了attrs数组存储属性。
    node.attr = attrs.reduce(function(pre, attr) {
      const name = attr.name;
      const value = attr.value;
      if (name == "class") {
        // 如果是class，单独存储为classStr
        node.classStr = value;
      }
      if (pre[name]) {
        if (Array.isArray(pre[name])) {
          pre[name].push(value); // 之前已经有相同的属性，则push
        } else {
          pre[name] = [pre[name], value]; //如果不是数组，转成数组存储。
        }
      } else {
        pre[name] = value; //如果不存在，则直接新增赋值。
      }
      return pre;
    }, {});
  }
  if (node.tag == "img") {
    node.imgIndex = wxImageArray.length; // 标识图片的索引。是整个图片里面第几张
    node.height = defaultHeight; // 微信图片需要高度，先默认一个高度
    const last = bufArray[bufArray.length - 1]; // 获取起始节点的最后一个节点
    let cls;
    if (last) {
      cls = last.classStr;
    }
    // const imgClass = ['feedback_mix_img'];我这里定义这个class的图片，是需要放大查看的，所以找出有这个class的图片
    cls &&
      imgClass.some(function(i) {
        if (cls.indexOf(i) != -1) {
          wxImageArray.push({ height: defaultHeight, src: node.attr.src });
          return true;
        }
        return false;
      });
  }
  if (unary) {
    // 如果是截止标签，后面不会有parseEndTag了，所以本个标签截止。
    pushChild(node); //将该起始标签Push为子节点
  } else {
    bufArray.unshift(node); // 未结束，所以需要保存起始标签，等结束标签之后，组装未一个完整的标签。
  }
}
```
将当前 node，加到 result 上的 child
```js
function pushChild(node) {
  if (bufArray.length === 0) {
    results.child.push(node);
  } else {
    const parent = bufArray[0];
    if (typeof parent.child == "undefined") {
      parent.child = [];
    }
    parent.child.push(node);
  }
}
```
HTMLParser 之后的截止标签回调。即处理到了截止标签。则需要将 push 的起始标签 push 到 AST 上去。  
如果标签不匹配，则说明匹配不对。但是起始标签也应该立即截止。比如<div\><span\></div\>，则最近的是 span，但是等来的结束标签确是 div，那么 span 应该立马截止。  
 我们记得上面的代码 parseEndTag，如果没有找到结束标签对应最近的起始标签，则会一直查找到最近的起始标签。所以碰到</div\>了，会触发两次 end，一个是 span，还有一个是 div。
```js
function end(tag) {
  const node = bufArray.shift();
  if (node.tag !== tag) console.error("invalid state: mismatch end tag");
  pushChild(node);
}
```
文本节点，直接 pushChild。
```js
function chars(text) {
  const node = {
    node: "text",
    text
  };
  pushChild(node);
}
```
经过以上处理之后，就会将 html 的字符串，转换为结构化的JSON对象。    
例子如下：
```html
<div class="feed_text" id="my_id">
  1111
  <p>2222</p>
</div>
```
经过HTMLParser之后，得到如下的json对象。
```js
{
    "node": "root",
    "child": [{
        "node": "element",
        "tag": "div",
        "attr": {
            "class": "feed_text",
            "id": "my_id"
        },
        "data": {},
        "type": "",
        "tagType": "block",
        "classStr": "feed_text",
        "child": [{
            "node": "text",
            "text": "1111"
        }, {
            "node": "element",
            "tag": "p",
            "attr": {
                "style": ""
            },
            "data": {},
            "type": "",
            "tagType": "block",
            "child": [{
                "node": "text",
                "text": "2222"
            }]
        }]
    }]
}
```
## 第二步：渲染 AST 语法树
将字符串转换为结构化的JSON对象之后，就完成了一大步，已经是结构化的数据了。最原先方案是使用小程序的component嵌套来处理，即每一个节点都对应一个component,这样的结果就是页面上面全是component的嵌套，测试结果性能非常差，节点数据一多，手势滑动都成问题，或许和component的实现有关吧。于是我们还是采取原始的template来实现。    
当然有人要说为啥不用小程序的rich-text组件。如果页面只是用来展示的话，直接用rich-text足以，但是我们这里，图片点击要放大，商品点击要跳转等事件需要处理，所以用rich-text达不到要求。    
这里是根目录的循环。一级节点
```html
<view class="rich_html_root">
    <block wx:for="{{parseDataIgn}}" wx:key="">
        <template is="wxParse0" data="{{item}}" />
    </block>
</view>
```
这里是我们已有的基本内容，比如商品组件，视频组件，投票组件，换行等。   
如果一旦检测到这些内容，整个子内容无需循环处理，直接渲染小程序的对应组件即可。
```html
<!-- 无层级，基础 -->
<template name="base">
    <rich-goods ref="rich-goods" wx:if="{{item.type=='goods'||item.type=='goods_list'}}" item-ign="{{item}}"></rich-goods>
    <!-- video类型 -->
    <gwq-video wx:elif="{{item.type == 'video'}}" ref="gwq-video" videourl="{{item.videourl}}" poster="{{item.poster}}"></gwq-video>
    <image wx:elif="{{item.type == 'face'}}" src="{{item.attr.src}}" class="{{item.classStr}}" lazy-load="{{true}}"/>
    <!-- img类型 -->
    <rich-image ref="rich-image" wx:elif="{{item.tag == 'img'}}" cls="{{item.classStr}}" src="{{item.attr.src}}" big-index="{{item.imgIndex}}" css="{{item.attr.style}}"></rich-image>
    <pk-vote ref="pk-vote" wx:if="{{item.type=='pk'||item.type=='vote'}}" item-ign="{{item}}"></pk-vote>
    <!-- br类型 -->
    <block wx:elif="{{item.tag == 'br'}}">
        <view class="{{item.classStr}}" style="{{item.attr.style||''}}"></view>
    </block>
</template>
```
循环模板，主要区分三大类：  
- li标签 : 用view+text渲染。  
- base标签 ：组件 + 基础元素节点
- 其他块级标签 : 用view渲染
- 内联标签 : 用text渲染
- 文本标签 : 用text渲染
```html
<!-- 循环模版 -->
<template name="wxParse0">
    <!-- 判断是否是标签节点 -->
    <block wx:if="{{item.node == 'element'}}">
        <!-- li类型 -->
        <block wx:elif="{{item.tag == 'li'}}">
            <view class="{{item.classStr}} rich_li">
                <view class="{{item.classStr}} rich_li_inner">
                    <view class="{{item.classStr}} rich_li_text">
                        <text class="{{item.classStr}}"></text>
                    </view>
                    <view class="{{item.classStr}} rich_li_text">
                        <block wx:for="{{item.child}}" wx:for-item="item" wx:key="">
                            <template is="wxParse1" data="{{item}}"></template>
                        </block>
                    </view>
                </view>
            </view>
        </block>
        <template is="base" wqvue-is="base" wx:elif="{{item.tag=='video'||item.tag=='img'||item.tag=='br'||item.type=='goods'||item.type=='video'||item.type=='goods_list'||item.type=='pk'||item.type=='vote'||item.type=='face' || item.tag=='style'}}" data="{{item}}" ></template>
        <!-- 其他块级标签 -->
        <block wx:elif="{{item.tagType == 'block'}}">
            <view class="{{item.classStr}} rich-{{item.tag}}" style="{{item.attr.style||''}}">
                <block wx:for="{{item.child}}" wx:for-item="item" wx:key="">
                      <template is="wxParse1" data="{{item}}"></template>
                </block>
            </view>
        </block>
        <!-- 内联标签 -->
        <text wx:else class="{{item.classStr}} rich-{{item.tag}} rich-{{item.tagType}}" style="{{item.attr.style||''}}">
            <block wx:for="{{item.child}}" wx:for-item="item" wx:key="">
                 <template is="wxParse1" data="{{item}}"></template>
            </block>
        </text>
    </block>
    <!-- 判断是否是文本节点 -->
    <block wx:elif="{{item.node == 'text'}}">
        <!-- 如果是，直接进行 -->
        <text class="rich_text {{item.text == '\\n' ? 'rich-hide':''}}">{{item.text}}</text>
    </block>
</template>
```
# 第三步：小程序图片的处理

小程序图片的渲染和点击放大查看大图处理。  
节点比较简单，渲染一个小程序image标签。增加load和tap事件即可。
```html
<image src="{{src}}" class="rich-image-com {{cls}}" bindload="imgLoaded" bindtap="imgTap" mode="{{mode}}"  lazy-load="{{true}}" style="width:{{width}};height:{{height}}px"/>
```
图片加载事件imgLoaded，如果是富文本的表情，则不需要处理点击放大，如果真实图片大于屏幕宽，则等比压缩显示。  
我这里富文本的大图都是宽度充满显示。则等比计算图片的高度即可。
```js
const winWidth = wx.getSystemInfoSync().windowWidth;
imgLoaded(e){
  let w = e.detail.width;
  let hh = e.detail.height;
  if (this.cls == 'face_image') {
      if (w > winWidth) {
          hh = (winWidth * hh) / w;
          w = winWidth;
      }
      this.width = w + 'px';
      this.height = hh;
  } else {
      const h = (winWidth * hh) / w;
      let n = 100;
      this.height = Math.ceil(h * n) / n;// 处理用户上传的图片但是系统自动裁剪的1px的缝隙问题。
  }
},
imgTap(){
  this.triggerEvent("imgTapForBigView",this.src,this.index);
}
```
在父组件上，监听到子组件图片的点击，然后拿到点击的图片的索引，然后使用wx.previewImage实现预览，bigImgIgn则在HTMLParser里面拿到的需要预览大图的图片数组。
```js
/**
 * 点击预览大图
 * @param {*} e
 * @param {*} that
 */
function richImgTap (that, src, index) {
    if (typeof wx != 'undefined' && wx.previewImage && that.data.bigImgIgn[index || 0] == src) {
        wx.previewImage({
            current: that.data.bigImgIgn[index || 0], // 当前显示图片的http链接
            urls: that.data.bigImgIgn // 需要预览的图片http链接列表
        })
    }
}
```
