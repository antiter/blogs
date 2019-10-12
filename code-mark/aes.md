# aes 的使用

使用 crypto-js 做 AES 的加密解密   


[https://github.com/brix/crypto-js](https://github.com/brix/crypto-js)    

详细 AES 介绍:[https://blog.csdn.net/qq_28205153/article/details/55798628](https://blog.csdn.net/qq_28205153/article/details/55798628)  

当前需要后台加密之后的文件，前端需要解密（显然被否决了，安全问题）   
先看下后台代码：  
```java
      public static String encryptPin(String pin) {
        try {
            // 加密
            byte[] encryptArray = AESUtil.encrypt(pin, DETAIL_PIN_KEY);
            // 编码
            return AESUtil.base64Encode(encryptArray);
        } catch (Exception e) {
            LOGGER.error("CpsPinUtil encryptPin Exception:{}", e);
        }
        return null;
    }
    public static String decryptPin(String pin) {
        try {
            // 解码
            byte[] decodeArray = AESUtil.base64Decode(pin);
            // 解密
            byte[] decryptArray = AESUtil.decrypt(decodeArray, DETAIL_PIN_KEY);
            return new String(decryptArray);
        } catch (Exception e) {
            LOGGER.error("CpsPinUtil decryptPin Exception:{}", e);
        }
        return null;
    }
   public static byte[] encrypt(String content, byte[] keyArray) {
        try {
            SecretKeySpec key = new SecretKeySpec(keyArray, "AES");
            Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
            cipher.init(1, key);
            byte[] byteContent = content.getBytes("UTF-8");
            return cipher.doFinal(byteContent);
        } catch (Exception var5) {
            LOGGER.error("使用AES加密失败 ", var5);
            return null;
        }
    }
     public static byte[] encrypt(String content, byte[] keyArray) {
        try {
            SecretKeySpec key = new SecretKeySpec(keyArray, "AES");
            Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
            cipher.init(1, key);
            byte[] byteContent = content.getBytes("UTF-8");
            return cipher.doFinal(byteContent);
        } catch (Exception var5) {
            LOGGER.error("使用AES加密失败 ", var5);
            return null;
        }
    }
    public static String base64Encode(byte[] bytes) {
        return Base64.encodeBase64String(bytes);
    }

    public static byte[] base64Decode(String base64Code) {
        return StringUtils.isBlank(base64Code) ? null : Base64.decodeBase64(base64Code);
    }
```

这里重点  AES/ECB/PKCS5Padding ：   

1. 代表的意思是使用 AES 加密
2. 使用 ECB 加密模式
3. 使用 PKCS5Padding 的补码方式，但是实际上 PKCS5Padding 对应的就是 PKCS7
4. DETAIL_PIN_KEY 就是加密秘钥了
5. cipher.init(1, key); 只用到了加密秘钥，没有用到偏移值

理解这个之后，我们就开始解密了。   

第一步是加密的秘钥，这个是加密的时候确定的，那么想要前端解密，也需要这个秘钥。

如果是一个字符串秘钥： 直接使用即可，如果是数组，则需要转换。   

```js
import CryptoJS from 'crypto-js';
function aesKeyBytes(value) {
    if(typeof value=='string'){
        return value;
    }
    var key_Int = new Int16Array([1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16]);// 随便写的
    var keyBytes = Int16Array(key_Int);
    return keyBytes;
}
function Int16Array(u8arr) {
    var len = u8arr.length;
    var words = [];
    for (var i = 0; i < len; i++) {
        words[i >>> 2] |= (u8arr[i] & 0xff) << (24 - (i % 4) * 8);
    }
    return CryptoJS.lib.WordArray.create(words, len);
  }
```
执行 aesKeyBytes  生成 key，由于后台没有使用到 iv（偏移值），所以可以设置为空即可。  
```js
    const decrypted =CryptoJS.AES.decrypt(encrypted,key,{
        iv:iv
        mode:CryptoJS.mode.ECB,
        padding:CryptoJS.pad.Pkcs7
    });
    return decrypted.toString(CryptoJS.enc.Utf8);     

```
至此，上面的已经解密完成了。    
我们再看下加密：   

```js
function getAesString(src,key,iv){//加密
    var encrypted =CryptoJS.AES.encrypt(src,key,{
        iv:iv
        mode:CryptoJS.mode.ECB,
        padding:CryptoJS.pad.Pkcs7
    });
    return encrypted.toString();    //返回的是base64格式的密文
}
```