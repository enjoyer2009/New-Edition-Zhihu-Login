---
title: Scrapy爬虫学习(二)
is_about: true
date: 2018-05-30 16:33:35
tags: python爬虫
---

# 1.Cookie和Session的区别

如果你登录知乎，填写过用户名、密码下次进来的时候不想再填写了，那么你在第一次登录后，服务器就会发送给你的浏览器一个Cookie，Cookie中包含了你的用户名、密码，下次再次发送请求给知乎的时候，浏览器会自动给请求加上Cookie，这样服务器就能知道你是谁。这就是Cookie的机制。

但是这种机制是不安全的，当你本地Cookie被别人获取后，就能直接使用你的账号了。于是出现了session机制：当你第一次以用户名、密码登录知乎时，知乎服务器会自己生成一条Session Id和sesson Value保存在服务器端，同时将Session Id作为Cookie中的一个键值对返回给浏览器，当你第二次请求知乎服务器的时候，请求会自动加上Session Id，那么服务器端收到Session Id后，会在服务器上查询是否有此Session Id,如果查询到，那么就匹配到相应的Session Value，也就是包含用户名密码的部分，这时候服务器就能识别出来你是那个用户了。

明确一点：Session Value中包含的是加密的用户名、密码等用户的Profile，是存放在服务器端，同事每一个Session Id都是有一个有效期的，这两点保证了安全性。

# 2.Http状态码
|code|说明|
|----|---------|
|200|请求被成功处理|
|301/302|永久重定向/临时重定向|
|403|没有访问权限|
|404|灭有对应的资源|
|500|服务器错误|
|503|服务器停机或正在维护|

# 3.分析知乎登录
## 3.1 找到登录请求URL
通过尝试手机号码和邮箱号，分析得出知乎统一登录网址为：https://www.zhihu.com/api/v3/oauth/sign_in
![登录请求URL](/home/liuke/Pictures/get_login_url.png)

## 3.2 找到登录时的header部分
```
:authority: www.zhihu.com
:method: POST
origin: https://www.zhihu.com
referer: https://www.zhihu.com/signup?next=%2F
user-agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36
x-xsrftoken: 405b7e07-d4f5-4b35-8b70-3e129d97a4d8
```
主要的是找到x-xsrftoken,以前的教程写的都是\_xsrf在返回的html中标签中，改版后变化了，我们可以在返回的cookie中找到，所以
首先设置一个header
```python
agent = "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"

header = {
    "HOST": "www.zhihu.com",
    "Referer": "https://www.zhihu.com/",
    "User-Agent": agent,
}
```
取得\_xsrf
```python
def get_xsrf():
	response = session.post("https://www.zhihu.com/signup?next=%2F", headers=header)
return response.cookies['_xsrf']
```
随后更新header
```python
    header.update({
        "X-Xsrftoken": xsrf
    })
```

## 3.3分析请求登录页面的data数据
![aasd](/home/liuke/Pictures/Selection_001.png)
### 3.3.1得到请求的data数据格式为下
```
------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="client_id"

c3cef7c66a1843f8b3a9e6a1e3160e20
------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="grant_type"

password
------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="timestamp"

1527727860453
------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="source"

com.zhihu.web
------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="signature"

8417b29b51739a7dba377934e7678c35625464f0
------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="username"

+8615639151994
------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="password"

admin123
------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="captcha"


------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="lang"

en
------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="ref_source"

homepage
------WebKitFormBoundary4YUIdOtklYTeomJn
Content-Disposition: form-data; name="utm_source"


------WebKitFormBoundary4YUIdOtklYTeomJn--
```
### 3.3.2 通过多次请求发现如下会改变的信息只有
```
username：
password：
timestamp：
captcha:
signature:
```
其中用户名、密码是自己控制的
#### 3.3.2.1 timestamp
比较简单就是距离1970年过去的秒数的整数部分。
```
timestamp = str(int(time.time()))
```
#### 3.2.2.2 captcha
验证码处理：现在知乎更新了，有的使用字母验证码，有的使用点击倒立的数字验证码。但是我发现登录时只考虑图形字母验证码也可以登录，但不知道*** 为什么 ***？

对字母验证码处理的方法就是下载图片，然后打开，进行手动输入

分析对验证码图片的请求url和header

**请求url为**：https://www.zhihu.com/api/v3/oauth/captcha?lang=cn

**header如下**：
```
    :authority: www.zhihu.com
    :method: GET
    :path: /api/v3/oauth/captcha?lang=cn
    :scheme: https
    accept: application/json, text/plain, */*
    accept-encoding: gzip, deflate, br
    accept-language: zh-CN,zh;q=0.9,en;q=0.8
    authorization: oauth c3cef7c66a1843f8b3a9e6a1e3160e20
    cookie: d_c0="AIDk5lsxrA2PTn6UE7w8ZXwIcLwr6s4V8TM=|1527673963"; q_c1=29e9198d965c42b4b4d13820bc7023db|1527673963000|1527673963000; _zap=a0bc8c13-50e7-484b-abde-db97010a065b; l_cap_id="YzIxYmFmNzg0YjViNGZmODljNTIwMjUwMTQ5NWY0NTY=|1527688563|f3bfccc25e61ab0e6c92295650f257e2f9cd779b"; r_cap_id="YTE3M2JlMWQ4NWQzNDA2NzllYThmMWYxNjMxZmRhMTY=|1527688563|be9b0cd7843bf090e5e0194ceb29a99fc60a1cce"; cap_id="ZWFjYTg3ODljMzI0NDVmOTgyYmE0NjRiMGQyZGRmYmU=|1527688563|4636e8decd51156afe879407f16e6c7f57e222ce"; tgw_l7_route=5bcc9ffea0388b69e77c21c0b42555fe; _xsrf=405b7e07-d4f5-4b35-8b70-3e129d97a4d8; capsion_ticket="2|1:0|10:1527727674|14:capsion_ticket|44:Y2M3YTcwYWQ3OGFiNGIxMjk3MWUwY2I5ZWQ0OWM0ZjQ=|ed0700eadd8466ffd6f4c61dfbce11a1f4d11483f32f1ae8262501a7fd859558"
    if-none-match: "fa4cf03c0ac47ca1c52ed2df2b71dfda86db6655"
    referer: https://www.zhihu.com/signup?next=%2F
    user-agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36
    x-udid: AIDk5lsxrA2PTn6UE7w8ZXwIcLwr6s4V8TM=
```
发现了有一个authorization字段，我在此踩坑，没加这个字段的header不能成功请求图片验证码
所以：


```python
def get_captcha():
    captcha_url = 'https://www.zhihu.com/api/v3/oauth/captcha?lang=cn'
    header.update({
        "authorization": "oauth c3cef7c66a1843f8b3a9e6a1e3160e20"
    })
    response = session.get(captcha_url, headers=header)
    r = re.findall('"show_captcha":(\w+)', response.text)
    if r[0] == 'false':
        return ''
    else:
        print("需要输入验证码！")
        response = session.put('https://www.zhihu.com/api/v3/oauth/captcha?lang=cn', headers=header)
        show_captcha = json.loads(response.text)['img_base64']
        with open('captcha.jpg', 'wb') as f:
            f.write(base64.b64decode(show_captcha))
        try:
            im = Image.open('captcha.jpg')
            im.show()
            im.close()
        except:
            print("打开文件失败！")

        captcha = input('输入验证码:')

        return captcha
```
#### 3.2.2.3 signature

这边是参照[大神的分析](https://blog.csdn.net/lifeifei1245/article/details/78959172)

多次请求,其他参数都是固定的,但是signature参数是个什么东西…知道意思是签名,但是我们要从哪里获取这个呢.网页源码里没有,那肯定是js生成的,去js里搜索在 
https://static.zhihu.com/heifetz/main.app.19b9c7c4c4502d8ef477.js 中总算是找到了.为了好看下载到编译器里,实在太大,编译器直接卡死了,太尴尬了….漫长的等待后拿到这么一段js
```javascript
function (e, t, n) {
    "use strict";
    function r(e, t) {
        var n = Date.now(), r = new a.a("SHA-1", "TEXT");
        return r.setHMACKey("d1b964811afb40118a12068ff74a12f4", "TEXT"), r.update(e), r.update(i.a), r.update("com.zhihu.web"), r.update(String(n)), c({
            clientId: i.a,
            grantType: e,
            timestamp: n,
            source: "com.zhihu.web",
            signature: r.getHMAC("HEX")
        }, t)
    }
```
可以看出是使用sha-1 key='d1b964811afb40118a12068ff74a12f4'和其他的一些字段生成的HMAX
```python
def get_signature(time_str):
    h = hmac.new(key='d1b964811afb40118a12068ff74a12f4'.encode('utf-8'), digestmod=sha1)
    grant_type = 'password'
    client_id = 'c3cef7c66a1843f8b3a9e6a1e3160e20'
    source = 'com.zhihu.web'
    now = time_str
    h.update((grant_type + client_id + source + now).encode('utf-8'))
    return h.hexdigest()
```

## 3.4 封装data和header进行登录

```python
def zhihu_login(account, password):
    # 知乎登录
    time_str = str(int(time.time()))
    xsrf = get_xsrf()
    header.update({
        "X-Xsrftoken": xsrf
    })
    post_url = "https://www.zhihu.com/api/v3/oauth/sign_in"
    post_data = {
        "client_id": "c3cef7c66a1843f8b3a9e6a1e3160e20",
        "grant_type": "password",
        "timestamp": time_str,
        "source": "com.zhihu.web",
        "signature": get_signature(time_str),
        "username": account,
        "password": password,
        "captcha": get_captcha(),
        "lang": "cn",
        "ref_source": "homepage",
        "utm_source": ""
    }

    response = session.post(post_url, data=post_data, headers=header, cookies=session.cookies)
    if response.status_code == 201:
        # 保存cookie，下次直接读取保存的cookie，不用再次登录
        print("登录成功")

        response = session.post("https://www.zhihu.com", headers=header, cookies=session.cookies)
       #print(response.text)
        session.cookies.save()
    else:
        print("登录失败")
```


如果登录成功我们要保存cookie，下次登录就直接实用cookie登录即可，所以我们首先要加载cookie

```python
session = requests.session()

session.cookies = cookielib.LWPCookieJar(filename="cookies.txt")  # cookie存储文件，


try:
    session.cookies.load(ignore_discard=True)  # 从文件中读取cookie
except:
    print("cookie 未能加载")
```
## 3.5 判断是否已经登录
因为我们加载了cookie，如果cookie有效且正确，这不用再次登录

这里采用点击用户个人中心的链接，且不允许跳转，从而获取返回的状态码，如果状态码为200则证明登录了，否则要进行登录
```python
def is_login():
    # 通过个人中心页面返回状态码来判断是否登录
    # 通过allow_redirects 设置为不获取重定向后的页面

    response = session.get("https://www.zhihu.com/inbox", headers=header, allow_redirects=False)

    print(response.cookies)
    print(response.status_code)
    if response.status_code != 200:
        print("尚未登录")
        zhihu_login("+***", "***")
    else:
        print("你已经登陆了")

```
至此，就可以成功登录啦！

具体实现完整代码，请移驾[github](https://github.com/liukekomorebi)


