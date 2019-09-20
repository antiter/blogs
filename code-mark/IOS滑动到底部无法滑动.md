
```js
dealIosScroll(cls){
    if(!cls||getOs()!='ios'){return;}
    var elem =  document.querySelector(cls);
    var startTopScroll = elem.scrollTop;
    //当滚动条在最顶部的时候
    if(startTopScroll <= 0)
        elem.scrollTop = 1;
    //当滚动条在最底部的时候
    if(startTopScroll + elem.offsetHeight >= elem.scrollHeight){
        elem.scrollTop = elem.scrollHeight - elem.offsetHeight -1;
    }
}
```