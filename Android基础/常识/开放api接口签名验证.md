# 开放api接口签名验证

来源:[微信公众号CodeL | http://www.daimali.com](http://www.cnblogs.com/codelir/p/5327462.html)

不要急，源代码分享在最底部，先问大家一个问题，你在写开放的API接口时是如何保证数据的安全性的？先来看看有哪些安全性问题在开放的api接口中，我们通过http Post或者Get方式请求服务器的时候，会面临着许多的安全性问题，例如：

* 请求来源(身份)是否合法？
* 请求参数被篡改？
* 请求的唯一性(不可复制)

为了保证数据在通信时的安全性，我们可以采用参数签名的方式来进行相关验证。

## 案列分析

我们通过给某 [移动端(app)] 写 [后台接口(api)] 的案例进行分析：     

* 客户端： 以下简称app   
* 后台接口：以下简称api


我们通过app查询产品列表这个操作来进行分析：

**app中点击查询按钮==》调用api进行查询==》返回查询结果==>显示在app中**

上代码啦 -_-！

## 一、不进行验证的方式

api查询接口：

![](2/1.png)

app调用：http://api.test.com/getproducts?参数1=value1.......

如上，这种方式简单粗暴，通过调用getproducts方法即可获取产品列表信息了，但是 这样的方式会存在很严重的安全性问题，没有进行任何的验证，大家都可以通过这个方法获取到产品列表，导致产品信息泄露。
那么，如何验证调用者身份呢？如何防止参数被篡改呢？

## 二、MD5参数签名的方式

我们对api查询产品接口进行优化：

* 1.给app分配对应的key、secret
* 2.Sign签名，调用API 时需要对请求参数进行签名验证，签名方式如下： 
   * a、按照请求参数名称将所有请求参数按照字母先后顺序排序得到：*keyvaluekeyvalue...keyvalue  字符串如：将arong=1,mrong=2,crong=3 排序为：arong=1, crong=3,mrong=2*  然后将参数名和参数值进行拼接得到参数字符串：arong1crong3mrong2。 
   * b、将secret加在参数字符串的头部后进行MD5加密 ,加密后的字符串需大写。即得到签名Sign


新api接口代码:

![](2/2.png)

app调用：http://api.test.com/getproducts?**key**=app_key&**sign**=BCC7C71CF93F9CDBDB88671B701D8A35&参数1=value1&参数2=value2.......

**注：secret 仅作加密使用, 为了保证数据安全请不要在请求参数中使用。**

如上，优化后的请求多了key和sign参数，这样请求的时候就需要合法的key和正确签名sign才可以获取产品数据。这样就解决了身份验证和防止参数篡改问题，如果请求参数被人拿走，没事，他们永远也拿不到secret,因为secret是不传递的。再也无法伪造合法的请求。


但是...这样就够了吗？细心的同学可能会发现，如果我获取了你完整的链接，一直使用你的key和sign和一样的参数不就可以正常获取数据了...-_-!是的，仅仅是如上的优化是不够的


### 请求的唯一性：

为了防止别人重复使用请求参数问题，我们需要保证请求的唯一性，就是对应请求只能使用一次，这样就算别人拿走了请求的完整链接也是无效的。

唯一性的实现：在如上的请求参数中，我们加入时间戳 ：**timestamp**（yyyyMMddHHmmss），同样，时间戳作为请求参数之一，也加入sign算法中进行加密。

新的api接口：

![](2/3.png)

app调用：
http://api.test.com/getproducts?**key**=app_key&**sign**=BCC7C71CF93F9CDBDB88671B701D8A35&**timestamp**=201603261407&参数1=value1&参数2=value2.......


如上，我们通过timestamp时间戳用来验证请求是否过期。这样就算被人拿走完整的请求链接也是无效的。


### Sign签名安全性分析：

通过上面的案例，我们可以看出，安全的关键在于参与签名的secret，整个过程中secret是不参与通信的，所以只要保证secret不泄露，请求就不会被伪造。

## 总结

上述的Sign签名的方式能够在一定程度上防止信息被篡改和伪造，保障通信的安全，这里使用的是MD5进行加密，当然实际使用中大家可以根据实际需求进行自定义签名算法，比如：RSA，SHA等。

相关方法源码分享：

![](2/4.png)

![](2/5.png)

咳咳！是的，就是图片！-_- 如果你觉得不开心，来来来，公众号CodeL留言我把源码全发给你！

来源：代码里 

链接：[http://www.daimali.com/index.php/2016/04/27/241/](http://www.daimali.com/index.php/2016/04/27/241/)


这里有讲解为什么要用timestamp+nonce的组合

[http://stackoverflow.com/questions/6865690/whats-the-point-of-a-timestamp-in-oauth-if-a-nonce-can-only-be-used-one-time](http://stackoverflow.com/questions/6865690/whats-the-point-of-a-timestamp-in-oauth-if-a-nonce-can-only-be-used-one-time)