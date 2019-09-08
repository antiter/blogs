# H5页面&小程序如何实现emoji表情？

emoji表情都非常熟悉了，比如微信的会话窗口可以发表情。   
但是仔细看有一个重要的发现，比如朋友给你发一个emoji表情，在聊天会话列表页查看最近消息，会发现有点不同，体现在如下：  

- 列表页看到的就是一个表情，比如大笑😄的表情。
- 列表页看到的是一个文字，比如[发呆]，[懵逼]

前者就是默认emoji表情了，后者则是自定义表情。显然你拿着微信的[懵逼]在微博上是显示不出微信的[懵逼]表情的。    
本文就讲述在H5和微信小程序下的这两种emoji表情的实现。   

## 自定义emoji表情

自定义，顾名思义就是自己定义的表情，不是通用的表情，按照自己系统的一套解析规则来实现的，只在我的系统里面能识别的。         
示例：  
![买买买](./images/emoji4h5_2.gif)  

### 表情定义

首先你需要定义自己的一套规则的表情，是绝对路径的表情。比如：  

```js
export default emojiData = {
    "买买买": ["1",true,1]
}
```  

这里我们约定表情图片有统一的路径格式，比如：https://wq.360buyimg.com/fd/wx/img/gwq/face/xxx.png。所以"买买买"实际上是对应https://wq.360buyimg.com/fd/wx/img/gwq/face/1.png

![买买买](./images/emoji4h5_1.png)  

这里定义了"买买买"这个表情文案对应的emoji表情的路径，当然自定义表情的文案和图片要一一对应上。   
为了便于区分用户普通文案，我们通常自定义表情都会统一加上"[ ]",比如“买买买”最终显示的时候就是"[买买买]"。为了保证之前已经使用过的表情能正常显示，在系统里面[买买买]永远都只能对应这个图片路径，这里定义了一个数组，后面两个参数我们后面解释。    

### 表情转文本

有了文案和图片的对应关系，按照业务需求，将表情渲染在表情面板上即可，这样在页面的表情面板就可以显示表情的图片路径，然后用户选择了某个表情，在文本框则对应显示图片的文案，比如[买买买]，当然存储在系统数据库的也是[买买买]文案。   

```js
"买买买": ["1",true,1] 
```
这里数组的true,表示当前表情是否还在使用，对于下线的表情，我们在渲染表情面板的时候要去除，但是已经生成的表情在页面中如果有还需要显示。   
数据的1，代表该表情显示在表情面板的第几屏，我这里第一屏是0，所以1代表显示在第二屏。   

```js
function getFaceList(panelIndex) {
    let arr = [];
    for (let txt in emojiData) {
        if (emojiData[txt][2] == panelIndex) { // 找到对应面板的显示。
            if (emojiData[txt][1]) {// 保证该表情未下线
                let s = getFaceUrl(emojiData[txt]);
                s = `<a href="javascript:;" title="${txt}"> <img data-src="${s}"></a>`;
                arr.push(s);
            }
        }
    }
    return arr;
}
```
这里是渲染表情面板，根据渲染的面板索引来渲染数据，只有该表情未下线才能渲染到面板里面。 

```js
function getFaceUrl(emoji) {
    return `https://wq.360buyimg.com/fd/wx/img/gwq/face/${emoji[0]}.png`
}
```

### 文本转图片

后台查询的文案当中可能含有表情，在页面显示的时候，我们就需要将文本转成图片。
```js
function text2pic(txt) {
    if (!txt) {
        return '';
    }
    txt = txt.replace(/\[([^\]]+)\]/g, function (v1, v2) {
        //v1： [买买买]   v2：买买买
        let d = emojiData[v2];
        return d ? '<img class="quan_icon_emoji" src="' + getFaceUrl(d) + '" />' : v1;
    });
    return txt;
}
```
这里将需要显示的文本全量搜索[xxx]的内容，然后和我们自定义的emoji表情去匹配，如果匹配上了,就说明是一个自定义表情，那么我们就输出一个img图片标签。   

当然这里也区别不了这个表情到底是你在面板选择的emoji[买买买]，还是你自行在文本框输入的文案[买买买]。你也会发现微信的[懵逼]实际上是区分表情和文本了，如果我们也需要这么做的话，在传入后台的时候就不能单单只是传入[买买买]这个文案了，比如\<span class="emoji1">[买买买]</span>，那我们之前的识别也需要按这个规则来了。有兴趣和需要的可以试下。  

> 由于这里将普通文本转换成了富文本，所以特别要注意xss的问题。在转换表情前做一次xss处理。确保不出现xss问题，然后在将富文本内容渲染到页面。


## 系统默认emoji表情

上述自定义表情可以解决我们系统当中需要有特殊意义的emoji表情，比如[买买买]的表情，就是我们业务的特殊表情，实际上并没有这个emoji。  

但是系统emoji要注意，它在不同的系统中可能显示的样式会有点差别，不过放心大笑终究是大笑，在哪个系统下都不会变成大哭，如果要求不那么高的感觉问题不大，只要知道有这个显示差异就行了。   
### Unicode码emoji表情

大笑😄的Unicode：\uD83D\uDE04   

在用户的输入法输入默认emoji表情，在存储后台的时候，我们需要处理下，将其转换成unicode码存储，比如调用方法emoji2Unicode来转换。   
```js
function emoji2Unicode (emoji) {
    let str = '';
    if (emoji && emoji.length > 0) {
        for (const char of emoji) {
            const index = char.codePointAt(0);
            if (index > 65535) {
                const h =
                    '\\u' +
                    (Math.floor((index - 0x10000) / 0x400) + 0xd800).toString(
                        16
                    );
                const c =
                    '\\u' + ((index - 0x10000) % 0x400 + 0xdc00).toString(16);
                str = str + h + c;
            } else {
                str += char;
            }
        }
    }
    return str;
}
```  
后台如果存储为unicode码的emoji了，返回到前端的时候，不需要特殊使用innerHTML或者vue的v-html来显示。当做普通文本显示即可。比如如下两种都可以显示为表情😄：  

```html
document.getElementById("emojiWrap").innerHTML='\uD83D\uDE04';
document.getElementById("emojiWrap").innerText='\uD83D\uDE04';
```  

> 如果我们调用后台是传参unicode码，比如\uD83D\uDE04，那么前端查询返回的时候，也需要是文本\uD83D\uDE04，这里后台数据库需要支持unicode字符存储，本文先不做讨论，有兴趣的可以翻阅。

### HTML实体存储emoji表情

除了使用unicode码存储emoji表情之外，我们还可以采取html实体存储的方式来实现。  

用户手机输入法输入表情之后，我们需要转换成实体存储。  

#### emoji ==>  html entity
```JS
function emoji2HtmlEntity (str) {
    const patt = /[\ud800-\udbff][\udc00-\udfff]/g;
    str = str.replace(patt, function (char) {
        let H, L, code;
        if (char.length === 2) {
            H = char.charCodeAt(0); // 取出高位
            L = char.charCodeAt(1); // 取出低位
            code = (H - 0xd800) * 0x400 + 0x10000 + L - 0xdc00; // 转换算法
            return '&#' + code + ';';
        } else {
            return char;
        }
    });
    return str;
}
```  
这里将表情转换为了实体存储，这个html entity对于数据库存储没有要求，只要能存储字符串就OK。比如如下例子：  

```JS
emoji2HtmlEntity('😄')  ==>  &#128516;
```
将这个html实体&#128516存储在后台，然后前端查询的时候，原样返回即可。

#### html entity ==> emoji

通常html实体我们也可以直接渲染，比如：  

```html
document.getElementById("emojiWrap").innerHTML='&#128516';
```

但是我们还是不能使用innerText来显示html entity，只会显示文本原样内容，另外在微信小程序中又不好显示了，所以我们可以统一采取将html entity转换成unicode码来方式来显示。  

如下：he.js  
```js
const regexInvalidEntity = /&#(?:[xX][^a-fA-F0-9]|[^0-9xX])/;
const regexDecode = /&(Gt|GT|ii);|&(gt|lt)(?!;)([=a-zA-Z0-9]?)|&#([0-9]+)(;?)|&#[xX]([a-fA-F0-9]+)(;?)|&([0-9a-zA-Z]+)/g;
const decodeMapNumeric = { '0': '\uFFFD', '128': '\u20AC', '130': '\u201A', '131': '\u0192', '132': '\u201E', '133': '\u2026', '134': '\u2020', '135': '\u2021', '136': '\u02C6', '137': '\u2030', '138': '\u0160', '139': '\u2039', '140': '\u0152', '142': '\u017D', '145': '\u2018', '146': '\u2019', '147': '\u201C', '148': '\u201D', '149': '\u2022', '150': '\u2013', '151': '\u2014', '152': '\u02DC', '153': '\u2122', '154': '\u0161', '155': '\u203A', '156': '\u0153', '158': '\u017E', '159': '\u0178' };
const invalidReferenceCodePoints = [1, 2, 3, 4, 5, 6, 7, 8, 11, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 127, 128, 129, 130, 131, 132, 133, 134, 135, 136, 137, 138, 139, 140, 141, 142, 143, 144, 145, 146, 147, 148, 149, 150, 151, 152, 153, 154, 155, 156, 157, 158, 159, 64976, 64977, 64978, 64979, 64980, 64981, 64982, 64983, 64984, 64985, 64986, 64987, 64988, 64989, 64990, 64991, 64992, 64993, 64994, 64995, 64996, 64997, 64998, 64999, 65000, 65001, 65002, 65003, 65004, 65005, 65006, 65007, 65534, 65535, 131070, 131071, 196606, 196607, 262142, 262143, 327678, 327679, 393214, 393215, 458750, 458751, 524286, 524287, 589822, 589823, 655358, 655359, 720894, 720895, 786430, 786431, 851966, 851967, 917502, 917503, 983038, 983039, 1048574, 1048575, 1114110, 1114111];

const stringFromCharCode = String.fromCharCode;

const object = {};
const hasOwnProperty = object.hasOwnProperty;
const has = function (object, propertyName) {
    return hasOwnProperty.call(object, propertyName);
};

const contains = function (array, value) {
    let index = -1;
    const length = array.length;
    while (++index < length) {
        if (array[index] == value) {
            return true;
        }
    }
    return false;
};

const merge = function (options, defaults) {
    if (!options) {
        return defaults;
    }
    const result = {};
    let key;
    for (key in defaults) {
        // A `hasOwnProperty` check is not needed here, since only recognized
        // option names are used anyway. Any others are ignored.
        result[key] = has(options, key) ? options[key] : defaults[key];
    }
    return result;
};

const parseError = function (message) {
    throw Error('Parse error: ' + message);
};
const codePointToSymbol = function (codePoint, strict) {
    let output = '';
    if ((codePoint >= 0xD800 && codePoint <= 0xDFFF) || codePoint > 0x10FFFF) {
        if (strict) {
            parseError('character reference outside the permissible Unicode range');
        }
        return '\uFFFD';
    }
    if (has(decodeMapNumeric, codePoint)) {
        if (strict) {
            parseError('disallowed character reference');
        }
        return decodeMapNumeric[codePoint];
    }
    if (strict && contains(invalidReferenceCodePoints, codePoint)) {
        parseError('disallowed character reference');
    }
    if (codePoint > 0xFFFF) {
        codePoint -= 0x10000;
        output += stringFromCharCode(codePoint >>> 10 & 0x3FF | 0xD800);
        codePoint = 0xDC00 | codePoint & 0x3FF;
    }
    output += stringFromCharCode(codePoint);
    return output;
};

const decode = function (html, options) {
    options = merge(options, decode.options);
    const strict = options.strict;
    if (strict && regexInvalidEntity.test(html)) {
        parseError('malformed character reference');
    }
    return html.replace(regexDecode, function ($0, $1, $2, $3, $4, $5, $6, $7, $8) {
        let codePoint;
        let semicolon;
        let decDigits;
        let hexDigits;

        if ($4) {
            // Decode decimal escapes, e.g. `&#119558;`.
            decDigits = $4;
            semicolon = $5;
            if (strict && !semicolon) {
                parseError('character reference was not terminated by a semicolon');
            }
            codePoint = parseInt(decDigits, 10);
            return codePointToSymbol(codePoint, strict);
        }

        if ($6) {
            // Decode hexadecimal escapes, e.g. `&#x1D306;`.
            hexDigits = $6;
            semicolon = $7;
            if (strict && !semicolon) {
                parseError('character reference was not terminated by a semicolon');
            }
            codePoint = parseInt(hexDigits, 16);
            return codePointToSymbol(codePoint, strict);
        }

        // If we’re still here, `if ($7)` is implied; it’s an ambiguous
        // ampersand for sure. https://mths.be/notes/ambiguous-ampersands
        if (strict) {
            parseError(
                'named character reference was not terminated by a semicolon'
            );
        }
        return $0;
    });
};
decode.options = {
    'isAttributeValue': false,
    'strict': false
};
export default decode;
```
这个的实现过程，我就不再阐述了，感兴趣的同学可以阅读下，还比较简单。  
在拿到文本之后，我们先用he处理下：  

```JS
import decode from './he';
text = decode(text);
```
这样又将html实体转换成能可直接显示的unicode码的emoji表情了，在小程序中也可以直接显示了。当然这个强大的库不仅仅只是转换emoji表情，可以参考： https://github.com/mathiasbynens/he


