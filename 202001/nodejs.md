# JavaScript

JavaScript 是一种属于网络的脚本语言，是一种解释性脚本语言，是一种运行在浏览器的，用来与HTML页面交互作用的语言。

# 编译器（Compiler）

就是将一种语言翻译成另一种计算机或虚拟机可以直接执行的语言（通常为机器语言，例如JAVA编译器，输出的是.class二进制文件，可被JVM解读）

# 机器语言（machine language）
机器语言是一种指令集的体系。这种指令集由二进制代码组成，称为机器码。它们可以被计算机的CPU直接解读并执行

# 编译型语言（Compiled language）

通过编译器产出的代码：如C、C++等

# 解释器

与编译器不同的是，解释器是在程序运行时，边运行，边转译，生成计算机可以执行的代码。
而编辑器则是预先就把计算机可直接执行的代码转译生成完了，生成计算机可以执行的代码    

JS 是一门解释性语言。边解释边运行，但是实际上还有一层预解析。    


# Node是什么

是一个基于 Chrome V8 引擎的 JavaScript 运行时。    
使用了一个事件驱动、非阻塞式 I/O 的模型。    
是一个让 JavaScript 运行在服务端的开发平台。     

V8 使用 C++ 开发，并在谷歌浏览器中使用。在运行 JavaScript 之前，相比其它的 JavaScript 的引擎转换成字节码或解释执行，V8 将其编译成原生机器码（IA-32, x86-64, ARM, or MIPS CPUs），并且使用了如内联缓存（inline caching）等方法来提高性能。有了这些功能，JavaScript 程序在 V8 引擎下的运行速度媲美二进制程序。


# 后台应用服务

这个是最基本的，也是用的最多的。操作redis，操作数据库，提供 web api服务。

# 直出服务

直接在服务端使用 js ，将数据和模板一起渲染成 HTML 页面结构返回浏览器。ejs,vue ssr等。

# 开发桌面应用程序

Node-Webkit 是NodeJS与WebKit技术的融合，提供一个跨Windows、Linux平台的客户端应用开发的底层框架，利用流行的Web技术 （Node.JS，JavaScript，HTML5）来编写应用程序的平台。

Electron 可以让你使用纯 JavaScript 调用丰富的原生(操作系统) APIs 来创造桌面应用。     
我们熟知的 vscode 就是 基于 Electron 开发的。   

# 操作系统

NodeOS 是采用NodeJS开发的一款友好的操作系统，该操作系统是完全建立在Linux内核之上的，并且采用shell和NPM进行包管理，采用NodeJS不 仅可以很好地进行包管理，还可以很好的管理脚本、接口等。

# 全栈工程师

全栈工程师是指掌握多种技能，并能利用多种技能独立完成产品的人。    
当前被很多 JS 开发人引用。是指会 前端 JS，会后端 Nodejs，会数据库的全端开发工程师。

# 大前端

大前端就是所有前端的统称，比如Android、iOS、web、Watch等。

# React Native

React Native (简称RN)是Facebook于2015年4月开源的跨平台移动应用开发框架，是Facebook早先开源的JS框架 React 在原生移动应用平台的衍生产物，目前支持iOS和Android两个平台。RN使用Javascript语言，类似于HTML的JSX，以及CSS来开发移动应用。

# 快应用

快应用是华为、小米、OPPO、魅族等国内9大主流手机厂商共同制定的轻量级应用标准。它也是采用JavaScript语言开发，原生控件渲染。

# Flutter

Flutter 是 Google推出并开源的移动应用开发框架，主打跨平台、高保真、高性能。开发者可以通过 Dart语言开发 App，一套代码同时运行在 iOS 和 Android平台。 Flutter提供了丰富的组件、接口，开发者可以很快地为 Flutter添加原生扩展。


# TensorFlow

TensorFlow 是一个端到端开源机器学习平台。它拥有一个包含各种工具、库和社区资源的全面灵活生态系统，可以让研究人员推动机器学习领域的先进技术的发展，并让开发者轻松地构建和部署由机器学习提供支持的应用。

# face-api.js

JavaScript face recognition API for the browser and nodejs implemented on top of tensorflow.js core

https://storage.googleapis.com/tfjs-models/demos/body-pix/index.html

https://mp.weixin.qq.com/s/LQ8mpNc5_8pU0y9LWfQXjw

https://tensorflow.google.cn/js/models?hl=zh_cn

https://github.com/justadudewhohacks/face-api.js

https://justadudewhohacks.github.io/face-api.js/face_expression_recognition

https://www.hksite.cn/prjs/christmashat/