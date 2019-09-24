
![canvas绘图](https://raw.githubusercontent.com/antiter/blogs/master/code-mark/images/canvas_1.png?raw=true)  


- 转换数据
- 加载需要渲染的图片
- canvas绘图
- 将base64生成链接图片
- postmessage给小程序

```js
function startDraw(){ 
    parseData(skuListTemp)
    .then((obj:DrawData)=>getImageDataAll(obj))
    .then((obj:DrawData)=>getCanvas(obj))
    .then((base64:string)=>parseShareImg(base64))
    .then(imageUrl=>{
        if(!imageUrl) return;
        dLog("drawShare",imageUrl);
        // @ts-ignore
        let config = window.shareConfig;
        let xcxData = {
            snsset:true,
            url:config.link,
            pageurl:config.link,
            title:config.title,
            imgurl:originImg(imageUrl).replace(/^(http:)?\/\//, 'https://')
        };
        postXcxShare(xcxData);
    }).catch((err)=>{
        dLog(err);
    });
}

```
将需要渲染的图片都加载完，这是异步过程，先Load
```js
function getImageDataAll (obj:DrawData){
     return new Promise((resolve, reject) => {
        Promise.all(obj.urls.map(item => getImageData(item))).then((urls)=>{
            obj.urls = urls;
            dLog("drawShare imageDone");
            resolve(obj)
        }).catch((err)=>{
            reject(err);
        });
    })
}
function getImageData(item){
    return new Promise((resolve, reject) => {
        const img = new Image();
        img.crossOrigin = 'anonymous';
        img.src = (item.img || '').replace(/^(http:)?\/\//, location.protocol+'//');
        img.onload = () => {
            item.width = img.width;
            item.height = img.height;
            item.img = img;
            resolve(item)
        }
        img.onerror = err => {
            reject(err)
        }
        img.onabort = img.onerror;
    })
}

```
小程序图片宽高5:4设计。
```js
function getCanvas(obj:DrawData) {
    return new Promise((resolve, reject) => {
        let myCanvas = document.getElementById('myCanvas');
        let width = "420";
        let height = "336";
        myCanvas.setAttribute("width", width);
        myCanvas.setAttribute("height",height);
        // @ts-ignore
        let canvas = myCanvas.getContext('2d');
        
        canvas.save();
        canvas.fillStyle = '#ffffff';
        canvas.fillRect(0, 0, width, height);
        canvas.fill();
        let paddingX=0;
        let paddingY = 0;
        drawImg(canvas,obj.urls,paddingX,paddingY,+width,+height);
        drawText(canvas,obj,paddingX,paddingY,+width,+height);
        dLog("drawShare canvasDone");
        // @ts-ignore
        resolve(myCanvas.toDataURL());
    });
}

```
urls就是图片了。Promise.all加载完之后，开始渲染图片。
```js
function roundedRect(ctx, x, y, width, height, radius) {
    ctx.strokeStyle = "#fff";
    ctx.beginPath();
    ctx.moveTo(x, y + radius);
    ctx.lineTo(x, y + height - radius);
    ctx.quadraticCurveTo(x, y + height, x + radius, y + height);
    ctx.lineTo(x + width - radius, y + height);
    ctx.quadraticCurveTo(x + width, y + height, x + width, y + height - radius);
    ctx.lineTo(x + width, y + radius);
    ctx.quadraticCurveTo(x + width, y, x + width - radius, y);
    ctx.lineTo(x + radius, y);
    ctx.quadraticCurveTo(x, y, x, y + radius);
    ctx.stroke();
}
function drawImg(canvas,urls,paddingX,paddingY,canvasW,canvasH){
    if(!urls||urls.length==0) return;

    let skuCount = 0;
    urls.map((d:{id:string,img:any,width,height,p})=>{
        if(d.id=='head'){
            canvas.save();
            canvas.arc(paddingX+20,paddingY+20,20,0, 2 * Math.PI);
            canvas.clip();
            canvas.drawImage(d.img, paddingX, paddingY, 40,40);
            canvas.restore();
        }else if(d.id=='bg'){
            let top =paddingY+20+40;
            let maxW = canvasW - 2*paddingX;
            let maxH = canvasH - top;
            canvas.save();
            canvas.beginPath();
            roundedRect(canvas, 0, top, maxW, maxH, 20);
            canvas.clip();  // 通过裁剪得到圆角矩形 
            if(d.height/d.width>maxH/maxW){// 高图
                let originH = d.height * maxW/d.width ;
                let clipTop = Math.round((originH - maxH)/2);
                canvas.drawImage(d.img,0,clipTop,d.width,d.height- clipTop*2,paddingX,top,maxW,maxH);
            }else{// 宽图
                let originW = d.width * maxH/d.height;
                let clipLeft = (originW - maxW)/2;
                canvas.drawImage(d.img,clipLeft,0,d.width-clipLeft*2,d.height,paddingX,top,maxW,maxH);
            }
            canvas.restore();
        }else if(d.id=='sku'){
            let x = canvasW-paddingX*2-20-80;
            let y = canvasH-paddingX*2-20-80 - (80+10)*skuCount;
            canvas.save();
            roundedRect(canvas, x, y, 80, 80, 10);
            canvas.clip();  // 通过裁剪得到圆角矩形 
            canvas.drawImage(d.img,x,y,80,80);
            canvas.fillStyle = '#ffffff';
            canvas.font = '20px Arial';
            if(d.p){
                d.p = String(d.p).replace(/\.00$/,"");
                let pw = canvas.measureText(d.p).width;
                canvas.fillText(d.p,x+(80-pw)/2,y+65);
            }
            canvas.restore();
            skuCount++;
        }
    });
}

```
```JS
context.drawImage(img,sx,sy,swidth,sheight,x,y,width,height);
```
swidth 可选。被剪切图像的宽度。   
sheight  可选。被剪切图像的高度。  

这里是裁剪之后的图片宽高，不是原图宽高，也不是需要渲染到画布的宽高，就是被裁剪了之后，该图片的宽高。


```js
function drawText(canvas,obj:DrawData,paddingX:number,paddingY:number,canvasW,canvasH){
    if(obj.userName){
        let str = getSubStr(obj.userName,10);
        if(str!=obj.userName){
            str+="...";
        }
        canvas.save();
        canvas.fillStyle = '#333333';
        canvas.font = '24px Arial';
        canvas.fillText(str,paddingX+10+40,paddingY+31);
        canvas.restore();
    }
    let startX=canvasW-paddingX;
    if(obj.viewCount){
        canvas.save();
        canvas.fillStyle = '#999999';
        canvas.font = '20px Arial';
        let c = obj.viewCount+"人观看";
        let cw = canvas.measureText(c).width;
        startX -=cw;
        canvas.fillText(c,startX,paddingY+32);
        canvas.restore();
    }
    
    //@ts-ignore
    let desc = window.shareConfig.desc;
    if(desc){
        canvas.save();
        canvas.fillStyle = '#ffffff';
        canvas.font = '24px Arial';
        let str = getSubStr(desc,20);
        if(str!=desc){
            str+="...";
        }
        canvas.fillText(str,paddingX + 20,canvasH-40);
        canvas.restore();
    }
    
    canvas.save();
    canvas.fillStyle = '#999999';
    canvas.font = '20px Arial';
    
    let c;
    let status = store.state.status;
    if(status ==1||status ==10){
        c = "直播中";
    }else if(status ==0){
        c="预告";
    }else if(status ==3){
        c = '回放';
    }
    if(c){
        startX -=10;
        canvas.save();
        canvas.beginPath();
        canvas.moveTo(startX,paddingY+18);
        canvas.lineTo(startX,paddingY+32);
        canvas.lineWidth = 2;
        canvas.strokeStyle = '#cccccc';
        canvas.closePath();
        canvas.stroke();
        
        startX -=10;
        let cw = canvas.measureText(c).width;
        startX = startX-cw;
        canvas.fillText(c,startX,paddingY+32);
        canvas.restore();
    }
    
    if(status==1){
        startX -=10;
        let arr = [6,10,12,4];
        canvas.save();
        arr.map((d)=>{
            canvas.beginPath();
            canvas.moveTo(startX,paddingY+32-d);
            canvas.lineTo(startX,paddingY+32);
            canvas.lineWidth = 2;
            canvas.strokeStyle = '#FF4142';
            canvas.closePath();
            canvas.stroke();
            startX -=4;
        });
        canvas.restore();
    }
}
function getSubStr(text:string,num:number):string{
    if(!text) return "";
    let start = 0;
    let result="";
    text.split("").some((str)=>{
        start += str.replace(/[\u00FF-\uFFFF]/g, '  ').length;
        if(start>num) return true;
        else result+=str;
        return false;
    });
    return result;
}
```