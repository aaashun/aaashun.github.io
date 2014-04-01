---
layout: post
title: 一种web service api授权设计
---

这段时间在设计web service api的安全授权机制, 目前给每个应用分配了appId和secretKey, 主要考虑两种情况: 1. 防篡改, 2. 防止重复请求

我们来先看一下目前常见的两种授权方式.

<h2> 微信的授权方式 </h2>

所有接口都是使用https协议

access\_token是公众号的全局唯一票据，公众号调用各接口时都需使用access_token。正常情况下access_token有效期为7200秒，重复获取将导致上次获取的access_token失效

获取access token

    request: HTTP GET https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=APPID&secret=APPSECRET
    response: {"access_token": "ACCESS_TOKEN", "expires_in": 7200} 

调用其它接口时需带上access_token参数

注意: 由于使用了https, 接口就简单多了, secret可以直接作为参数, 不用担心被监听或中间人攻击的问题

<h2> 二、盛大云的授权方式 </h2>

所有接口都是使用http协议, 需对每个请求作签名

使用HMAC(k, m) = H((k XOR opad) + H((k XOR ipad) + m)), http://www.ietf.org/rfc/rfc2104.txt

HMAC较常用的朴素的hash算法H(k + m)更安全

签名过程如下:

1.构造待签名的utf-8字符串

  m = HTTPVerb + "\n" + host + "\n" + uri + "\n" + querystring

  * HTTPVerb: http方法包括GET, POST, PUT, DELETE等
  * host: 小写服务器域名, 例如api.ku6vms.com
  * uri: 资源uri, 例如/video
  * querystring: 查询参数, 查询参数应按参数名升序串联, 无值参数不需传递, 多个值采用","隔开, 参数值需urlencode

2.计算签名值

    d = HMAC(k, m)
        k是SecretKAccessKey
        m是上面的待签名的utf-8字符串
        d是签名值

3.示例

    SndaAccessKeyId: DRINDPIN54JTQWCG3ZJS70QYW
    SecretAccessKey: NWYxMDhmOTctMjYyMS00NTI3LWJhODQtMTM2MDBjODlmM2Ji

  * 请求方式: POST
  * 请求地址: http://api.ku6vms.com/video
  * 请求参数:

    + 授权参数 - AUTHPARAMS, 传递方式都是query

      - Version 可选, 接口版本号
      - SndaAccessKeyId, 必填
      - Timestamp 必填, 操作时间戳iso8601
      - Expires 必填, 过期时间戳
      - Signature 必填, 签名值
      - SignatureMethod 必填, 签名方法有HmacSHA1, HmacSHA256
      - SignatureVersion 可选,填名算法的版本号

    + 接口参数

      - Title 必填, 视频标题
      - Cid 可选
      - Tag 可选
      - Description 可选
      - BypassEncoding 可选

  * 响应结果

    + requestId 请求的ID号
    + vid 新视频的ID号
    + sid 视频上传ID号
    + uploadUrl 视频的上传地址


例如

*请求

    http://api.ku6vms.com/video?Title=my-video&SndaAccessKeyId=DRINDPIN54JTQWCG3ZJS70QYW&Timestamp=2014-01-07T00%3A00%3A00&Expires=2014-01-07T00%3A01%3A00&Signature=xxxxxx&SignatureMethod=HmacSHA256

*响应

    <CreateVideoResponse>
      <requestId>514e1127-7f6a-4b72-af72-efff59f65b62</requestId>
      <sid>1363519061820</sid>
      <vid>170698</vid>
      <uploadUrl>http://58.215.38.23/vmsUpload.htm</uploadUrl>
    </CreateVideoResponse>



AWS与盛大云的授权机制是一样的, 连参数名都是一样的, 见[using query api] [1], [signature version 2] [2]
  [1]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-query-api.html#query-authentication
  [2]: http://docs.aws.amazon.com/general/latest/gr/signature-version-2.html

但是目前AWS用的是https, 盛大云还是http, 导致盛大云的接口授权机制是有问题的, 不能解决上边提到的请求重复发送问题和中间人篡改.

* 重复发送问题 比如同样的请求同时发两遍, 服务端光靠Timestamp等参数是没法发现这种问题的
* 中间人篡改 这里只是对query string作了hmac-sha1签名, 如果body里的内容被篡改了, 服务端也是不知道的

针对重复发送问题可以在AUTHPARAMS里加一个AuthId, AuthId是全局唯一的一个ID, 这样服务端就可以区分不同请求了

针对中间人篡改的问题, 可以把http content也作为HMAC待签名数据的一部分, 即m = querystring + content(费这么多事不如直接上https)

<h2> 综述 </h2>

如果有条件使用https, 则推荐类似微信的授权机制, 否则退而求其次使用类似盛大云的改进的接口授权机制.
