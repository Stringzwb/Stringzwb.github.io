---
layout: post
title: '一个bug彻底搞懂session和cookie工作原理'
date: 2023-9-19
author: zwb
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: java 网络知识
---

> 上周，我遇到了我在鱼厂的第一个大bug<br />

大家好，好几天没写博客了，因为我在被一个很恶心的bug所困扰。我给知识星球发自动回复的评论请求，神奇的发现了下面的问题：
>我使用hutool的httpUtils向知识星球发送请求，意外的发现第一次请求是成功的，第二次请求却报错 401，于是，折磨就开始了。<br />

让我们先来看一下错误代码：<br />
```java
HttpRequest httpRequest = HttpRequest.post(url)
    .header("Connection", "close")
    .header("Access-Control-Request-Headers","content-type,x-request-id,x-signature,x-timestamp,x-version")
    .header("Accept","application/json, text/plain, */*")
    .header("cookie", cookie)
    .header("user-agent", USER_AGENT)
    .header("x-request-id", requestId)
    .header("x-signature", signature)
    .header("x-timestamp", timestamp)
    .body(json);
```





在问题出现的第一天，我通过各种搜索引擎，chatgpt，呼叫群友，尝试了包括但不限于的以下方法，未果：<br />
- 0. 加断点后，得到401错误码
- 1. 暂时删掉 `X-signare` 无效
- 2. 查找答案帖子和 chatgpt，询问他人后，尝试清除缓存，仍无效
- 3. 添加完整请求头，无效
- 4. 添加完整 `cookie`，有效，但该解决方案用户操作不方便，驳回

困扰了我两三天后，我终于忍不住让我的同时帮助我解决。
同时修改后的代码
```java
OkHttpClient client = new OkHttpClient().newBuilder()
                .build();
        MediaType mediaType = MediaType.parse("application/json");
        RequestBody body = RequestBody.create(mediaType, json);
        Request realRequest = new Request.Builder()
                .url(url)
                .method("POST", body)
                .addHeader("accept", "application/json, text/plain, */*")
                .addHeader("cookie", cookie)
                .addHeader("user-agent", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36")
                .addHeader("x-request-id", requestId)
                .addHeader("x-signature", signature)
                .addHeader("x-timestamp", timestamp)
                .build();
        Response response = null;
        String bodyStr = "";
        try {
            response = client.newCall(realRequest).execute();
            bodyStr = response.body().string();

        } catch (IOException e) {
            throw new BusinessException(ErrorCode.OPERATION_ERROR);
        }

        System.out.println(bodyStr);
        return JSONUtil.toBean(bodyStr, CommentResponse.class);
    }
```

卧槽，这不就是换了个工具类吗？但仔细想想我的请求头参数完全一样，为什么okhttp3可以实现，而hutool却不行呢？于是，我又开始了探索。
探索的第一步是，我直接去了hutool的源码仓库，查看了很多issue。大家对httpUtils的意见很多。
![image](https://github.com/Stringzwb/Stringzwb.github.io/assets/103347797/c4165349-b8dd-4f03-b87c-6b92e1df3021)
搜索了大量issue中的问题后发现并没有什么卵用，于是！我就开始加断点地毯式搜索问题，果然，我发现在第一次发送请求后，响应头中出现了set-cookie字段，刚好401是和权限相关的，我想问题的原因应该就和cookie有关。
![74c486c888e04987c8c9d88ba185cec](https://github.com/Stringzwb/Stringzwb.github.io/assets/103347797/cbbd3806-3990-4ab5-877e-3152418660d0)
而且，两种工具类的返回内容并不相同，对于okhttp3，每次都会返回不同的sessionId，而hutool第一次返回了sessionId,后面报错401的时候却不返回了。
这时鱼皮给了我建议，现在你只能看到你请求的数据，但你不知道响应端拿到的cookie是什么，这时候你应该怎么做呢？没错，自己写一个服务端不就好了：
```java
@RestController
@RequestMapping("/test")
public class TestController {

    @PostMapping("/setCookie")
    public ResponseEntity<String> setCookie(HttpServletRequest request, @RequestBody String requestBody) {
        // 从请求头获取Cookie
        String cookiesFromHeader = request.getHeader("Cookie");
        System.out.println("Received Cookies from header: " + cookiesFromHeader);


        // 从HttpServletRequest对象获取Cookie
        javax.servlet.http.Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (javax.servlet.http.Cookie cookie : cookies) {
                System.out.println("Received Cookie from request object: " + cookie.getName() + "=" + cookie.getValue());
            }
        }

        HttpHeaders headers = new HttpHeaders();
        headers.put(HttpHeaders.SET_COOKIE, Collections.singletonList("zsxqsessionid=newSessionIdValue"));

        return ResponseEntity.ok()
                .headers(headers)
                .body("Cookie has been set");
    }

}
```
分别用两种工具进行测试
```java
public class NetTest {


    public static void main(String[] args) {
//        sendRequestWithOkHttp();
        sendRequestWithHutool();
    }

    public static void sendRequestWithOkHttp() {
        JsonObject json = new JsonObject();
        json.addProperty("key1", "value1");
        json.addProperty("key2", 42);
        json.addProperty("key3", true);

        OkHttpClient client = new OkHttpClient();
        MediaType mediaType = MediaType.parse("application/json");
        RequestBody requestBody = RequestBody.create(mediaType, json.toString());

        String url = "http://localhost:8081/test/setCookie";
        Request.Builder requestBuilder = new Request.Builder()
                .url(url)
                .method("POST", requestBody)
                .addHeader("accept", "application/json, text/plain, */*")
                .addHeader("cookie", "existingCookie=existingValue")
                .addHeader("user-agent", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36")
                .addHeader("x-request-id", "sdufisdgfs")
                .addHeader("x-signature", "sudhfisduf")
                .addHeader("x-timestamp","ss");

        for (int i = 1; i <= 2; i++) {
            try (Response response = client.newCall(requestBuilder.build()).execute()) {
                if (response.isSuccessful()) {
                    System.out.println("OkHttp - Request " + i + " successful");
                    System.out.println("Response body: " + response.body().string());
                } else {
                    System.out.println("OkHttp - Request " + i + " failed: " + response.message());
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void sendRequestWithHutool() {
        Map<String, Object> jsonMap = new HashMap<>();
        jsonMap.put("key1", "value1");
        jsonMap.put("key2", 42);
        jsonMap.put("key3", true);

        HttpRequest request = HttpRequest.post("http://localhost:8081/test/setCookie")
                .header("accept", "application/json, text/plain, */*")
                .header("user-agent", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36")
                .header("x-request-id", "sdufisdgfs")
                .header("x-signature", "sudhfisduf")
                .header("x-timestamp", "ss")
                .header("Cookie", "existingCookie=existingValue")
                .body(JSONUtil.toJsonStr(jsonMap));

        for (int i = 1; i <= 2; i++) {
            HttpResponse response = request.execute();
            if (response.isOk()) {
                System.out.println("Hutool - Request " + i + " successful");
                System.out.println("Response body: " + response.body());
            } else {
                System.out.println("Hutool - Request " + i + " failed: " + response.getStatus());
            }
        }
    }
}
```
返回的结果
okhttp3：
![image](https://github.com/Stringzwb/Stringzwb.github.io/assets/103347797/8d3ce6ed-378e-4c0a-b2c5-353f0841680e)

hutool 的 httpRequest:
![image](https://github.com/Stringzwb/Stringzwb.github.io/assets/103347797/45ab6cb3-80ba-4f1c-92b6-6f745d933713)
问题很明显，okhttp3不会主动保存返回的cookie内容（这里我查阅了资料，其实okhttp3也是可以保存cookie内容的，但是需要手动设置[cookieJAR](https://blog.csdn.net/weixin_45055749/article/details/122635365)），但hutool的工具类是会的，所以说两种工具类对cookie的处理是有区别的。
那么为什么okhttp3每次会返回不同的sessionId呢？而我在测试hutool工具类的时候为什么又不返回新的sessionId了呢？
【这里我真的是吃了**基础不好**的亏啊】
我们回顾一下session的工作原理。
>在相同浏览器下访问经过负载均衡的两个tomcat服务器，会出现什么情况？
>第一次：访问Tomcat-A，Tomcat-A发现没有此次会话的JSESSIONID，然后重新创建了一个JSESSIONID，并通过cookie返回；
>第N次：访问Tomcat-B，Tomcat-B同样会发现这个JSESSIONID没有相应的session信息，OK，就会再次创建一个JSESSIONID，并返回浏览器。
>第N+1次：再次访问Tomcat-A的时候，发现在服务器上的找不到JSESSIONID相应的session，但是对用户来说确是一次未退出的访问。
以下是Web会话的工作流程：

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Server as 服务器

    Client->>Server: 1、发送请求
    Server->>Server: 2、生成sessionid并存储在cookie
    Server->>Client: 发送cookie
    Client->>Client: 3、保存cookie
    Client->>Server: 4、发送带有cookie的后续请求
    Server->>Server: 从请求头中提取sessionid
    Server->>Server: 5、查找会话
    alt 找到会话
        Server->>Client: 发送存储的数据
    else 没有找到会话
        Server->>Server: 创建新的会话并存储在cookie
        Server->>Client: 6、发送新的cookie
    end
    Client->>Client: 7、结束会话并删除cookie
```
