---
layout:     post
title:      ppmessage学习：一
date:       2016-10-09 09:14:35
summary:    初步理解ppmessage处理流程，扩展ppcom
categories: ppmessage
thumbnail: fa-first-order
tags:
 - ppmessage
 - angularjs
 - python
 - tornado
 - websocket
---

# 理解ppmessage：一

看了两天ppmessage的源代码。对于毫无前台经验的我来说确实理解起来很吃力。好在在python端后台代码中，ppmessage的作者有打log处理。

{% highlight python %}
def _main():
    logging.getLogger().setLevel(logging.DEBUG)
    if len(sys.argv) == 1:
        ppmessage.backend._main()
{% endhighlight %}

所以在入口将log级别设为DEBUG，再在界面上通过ppcom、ppkefu对话，看log都有哪些处理。


## 大体流程

前台通过app_key、app_secret取得token，之后通过websocket做信息收发，websocket是事件触发，前台js也有订阅、发布处理结构。websocket在open时也有auth。
下面结合ppcom的js代码学习一下ppmessage的动作流程。


### 获取token

如图

![ppcom-auth]({{ site.baseurl }}/images/ppcom-auth.png)

对应的js代码如下（ppmessage/ppcom/src/controller/service/api.js - 105~130）

{% highlight javascript %}
this.getPPComToken = function(success, fail) {
    var urlPath = Configuration.auth + "/token";
    var requestData = "client_id=" + _apiKey + "&client_secret=" + _apiSecret + "&grant_type=client_credentials"
    
    $.support.cors = true;
    $.ajax({
        url: urlPath,
        type: 'post',
        data: requestData,
        headers: {
            "Content-Type": "application/x-www-form-urlencoded",
        },
        cache: false,
        crossDomain : true,
        beforeSend: function() {
            _onBeforeSend(urlPath, requestData);
        },
        success: function(response) {
            _apiToken = response.access_token;
            _onApiSuccess(response, success);
        },
        error: function(jqXHR, textStatus, errorThrown) {
            _onError(textStatus, fail);
        }
    });
};
{% endhighlight %}

这个界面有api_key、api_secret

![ppcom-key-secret]({{ site.baseurl }}/images/ppcom-key-secret.png)

至此，第一步清晰。通过restful调试工具，可以取到token。后台处理为（ppmessage/ppauth/tokenhandler.py - 103~109）

{% highlight python %}
self.set_header("Content-Type", "application/json")
self._header()
_return = {
    "access_token": _api_token,
    "token_type": "Bearer",
}
self.write(json.dumps(_return))
{% endhighlight %}


### WebSocket链接建立

前台核心消息收发都建立在websocket上，而且处理逻辑里有如果websocket发送失败，会调http接口发送。
如下图，为websocket的auth步骤

![ppcom-ws-auth]({{ site.baseurl }}/images/ppcom-ws-auth.png)

对应js代码如下（ppmessage/ppcom/src/controller/service/.js - 17~53）

{% highlight javascript %}
function auth () {
    var $api = Service.$api,
        $json = Service.$json,
        wsSettings = $notifyService.getWsSettings(),

        // auth params
        api_token = $api.getApiToken(),
        user_uuid = wsSettings.user_uuid,
        device_uuid = wsSettings.device_uuid,
        app_uuid = wsSettings.app_uuid,
        is_service_user = false,
        extra_data = {
            title: document.title,
            location: ( ( function() { // fetch `window.location`
                var loc = {};
                for (var i in location) {
                    if (location.hasOwnProperty(i) && (typeof location[i] == "string")) {
                        loc[i] = location[i];
                    }
                }
                return loc;
            } )() )
        };

    // register webSocket
    $notifyService.write($json.stringify({
        type: AUTH_TYPE,
        api_token: api_token,
        app_uuid: app_uuid,
        user_uuid: user_uuid,
        device_uuid: device_uuid,
        extra_data: extra_data,
        is_service_user: is_service_user
    }));
}
{% endhighlight %}

我用python的websocket-client库调了一下，部分代码如下

{% highlight python %}
def _on_open(ws):
    # 这里的内容是从log里扣出来的
    ws.send(
        '{"type":"auth","is_service_user":true,"api_token":"YWU3MjM2YWVjOGYxY2M5NDNlZmI5MWNjY2I4ZGM1NGY2Zjk2YzA5Mg==","user_uuid":"595f7791-8de0-11e6-a3fc-0242ac120004","device_uuid":"49ccc251-8de1-11e6-b3f1-0242ac120004","app_uuid":"597497ee-8de0-11e6-a8ec-0242ac120004"}')


def dive_websocket():
    websocket.enableTrace(True)
    ws = websocket.WebSocketApp("ws://172.16.60.128:8945/pcsocket/WS",
                                on_message=_on_message,
                                on_error=_on_error,
                                on_close=_on_close,
                                on_open=_on_open)
    ws.run_forever()
{% endhighlight %}

因为是从log里扣出来的内容，所以代码运行后，把登录的ppkefu给踢下来了。

输出结果如图

![ppcom-ws-auth-debug]({{ site.baseurl }}/images/ppcom-ws-auth-debug.png)


### 通过websocket发送订阅

**未完待续**


接下来的过程还要调用api进行创建设备、创建用户、创建回话等。之后通过websocket订阅消息事件，on_message函数为收到信息事件的毁掉函数。理论上这么一个流程，基本上就能通过编写代码收发消息了。如果跟微信的消息api对接，理论上可以将微信“伪装”成ppcom。
目前还局限于基于代码的猜测性理解，还需要调试验证。**后续更新**。
