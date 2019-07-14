---
layout: post
title:  "jsbridge"
categories: js
---

## jsbridge 相互调用原理

js 调用 java：reload iframe 回调 shouldOverrideUrlLoading 使用协议用过 url 拿到数据。eg：

<!-- more -->

```
yy://return/_fetchQueue/[{"data":{"id":1,"content":"这是一个图片 test\r\nhahaha"},"callbackId":"cb_1_1495693363960"}]
```

js 调用 java 回调：
1. js.send --> responseCallbacks[callbackId] = responseCallback;
2. reload iframe --> webView.flushMessageQueue() --> loadUrl(jsUrl, new CallBackFunction() {... --> responseCallbacks.put(BridgeUtil.parseFunctionName(jsUrl), returnCallback);
3. handlerReturnData() --> f= responseCallbacks.get(functionName); f.onCallBack(data); (「f」2 中 new CallBackFunction() {...)
4. defaultHandler.handler(m.getData(), responseFunction);  --> responseFunction.onCallBack("DefaultHandler response data");
5. esponseFunction.onCallBack --> queueMessage(responseMsg); --> dispatchMessage(m);
6. this.loadUrl(javascriptCommand); --> js.&#95;dispatchMessageFromNative(messageJSON) 完成回调。

7. 自己注册 BridgeHandler，js 使用「submitFromWeb」获取到对应的 handler 完成 js 调用 java。

```
webView.registerHandler("submitFromWeb", new BridgeHandler() {

  @Override
  public void handler(String data, CallBackFunction function) {
    Log.i(TAG, "handler = submitFromWeb, data from web = " + data);
            function.onCallBack("submitFromWeb exe, response data 中文 from Java");
  }

});

```

java 调用 js：webview.loadUrl(url) 调用 js 方法并通过 url 传递参数。eg：

```
javascript:WebViewJavascriptBridge._handleMessageFromNative(
  '{\"responseData\":\"DefaultHandler response data\",\"responseId\":\"cb_4_1495699731002\"}');
```

java 调用 js 回调：

html 页面点击按钮，调用 js testClick()，获取数据然后调用 WebViewJavascriptBridge.send（data,responseCallback） 发送请求。

```
function testClick() {

    // 获取 html 输入框的值
    var str1 = document.getElementById("text1").value;
    var str2 = document.getElementById("text2").value;

    //send message to native
    var data = {id: 1, content: "这是一个图片 test\r\nhahaha"};
    window.WebViewJavascriptBridge.send(
        data
        // function 匿名 js 的回调函数
        ,function(responseData) {
            // responseData native 返回给 html 的数据，并在 html 上展示。
            document.getElementById("show").innerHTML = "repsonseData from java, data = " + responseData
        }
    );

}

```

**把 message 加入 js 的 sendMessageQueue 队列，responseCallback 加入 responseCallbacks 中，并 reload iframe。**

```
//sendMessage add message, 触发native处理 sendMessage
function _doSend(message, responseCallback) {
    if (responseCallback) {
        // responseCallback 不为 null 时，以时间和 uniqueId 自增生成 callbackId ，加入 responseCallbacks 数组
        // 并为 message.callbackId 赋值 id
        var callbackId = 'cb_' + (uniqueId++) + '_' + new Date().getTime();
        responseCallbacks[callbackId] = responseCallback;
        message.callbackId = callbackId;
    }

    sendMessageQueue.push(message);
    messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
}

```

**native 不主动调用 _fetchQueue，通过 reload iframe 触发 native shouldOverrideUrlLoading 回调，拿到 url 中 js 传递的 message gson**

```
// 提供给native调用,该函数作用:获取sendMessageQueue返回给native,由于android不能直接获取返回的内容,所以使用url shouldOverrideUrlLoading 的方式返回内容
function _fetchQueue() {
    var messageQueueString = JSON.stringify(sendMessageQueue);
    sendMessageQueue = [];
    //android can't read directly the return data, so we can reload iframe src to communicate with java
    messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://return/_fetchQueue/' + encodeURIComponent(messageQueueString);
}

```

由 js reload iframe 触发 shouldOverrideUrlLoading。
webView.flushMessageQueue() 刷新消息队列，native 调用 &#95;fetchQueue() messagingIframe.src 进行赋值。
webView.handlerReturnData(url) 通过 url 携带数据，使用自定义协议 yy://return/&#95;fetchQueue/ 拿到返回数据。
data 由 id content callbackId 三部分组成。

```
@Override
public boolean shouldOverrideUrlLoading(WebView view, String url) {
    try {
        url = URLDecoder.decode(url, "UTF-8");
    } catch (UnsupportedEncodingException e) {
        e.printStackTrace();
    }

    // yy://return/_fetchQueue/[{"data":{"id":1,"content":"这是一个图片 test\r\nhahaha"},"callbackId":"cb_1_1495693363960"}]
    if (url.startsWith(BridgeUtil.YY_RETURN_DATA)) { // 如果是返回数据
        webView.handlerReturnData(url);
        return true;
    } else if (url.startsWith(BridgeUtil.YY_OVERRIDE_SCHEMA)) { //
        webView.flushMessageQueue();
        return true;
    } else {
        return super.shouldOverrideUrlLoading(view, url);
    }
}

```

webView.flushMessageQueue() 刷新消息队列，native 调用 &#95;fetchQueue()。
`jsUrl`:javascript:WebViewJavascriptBridge.&#95;fetchQueue() ,并添加了一个 &#95;fetchQueue 对应的 native returnCallback。

```
loadUrl(jsUrl, new CallBackFunction() {...

public void loadUrl(String jsUrl, CallBackFunction returnCallback) {
  this.loadUrl(jsUrl);
  responseCallbacks.put(BridgeUtil.parseFunctionName(jsUrl), returnCallback);
}

```


```
```

通过 url 拿到方法名 &#95;fetchQueue 得到对应的 native CallBackFunction 进行执行 f.onCallBack(data); 并删除 responseCallbacks 中的 CallBackFunction。

```
url: yy://return/_fetchQueue/[{"data":{"id":1,"content":"这是一个图片 test\r\nhahaha"},"callbackId":"cb_1_1495693363960"}]

void handlerReturnData(String url) {
  String functionName = BridgeUtil.getFunctionFromReturnUrl(url);
  CallBackFunction f = responseCallbacks.get(functionName);
  String data = BridgeUtil.getDataFromReturnUrl(url);
  if (f != null) {
    f.onCallBack(data);
    responseCallbacks.remove(functionName);
    return;
  }
}

```

`data`:

```
[{"data":{"id":1,"content":"这是一个图片 test\r\nhahaha"},"callbackId":"cb_1_1495693363960"}]
```

f.onCallBack(data);
callbackId 不为空，创建 responseFunction 把 responseMsg 加入 queueMessage。
defaultHandler.handler(m.getData(), responseFunction);

```
if (!TextUtils.isEmpty(callbackId)) {
  responseFunction = new CallBackFunction() {
    @Override
    public void onCallBack(String data) {
      Message responseMsg = new Message();
      responseMsg.setResponseId(callbackId);
      responseMsg.setResponseData(data);
      queueMessage(responseMsg);
    }
  };
}

// m.getData() {"id":1,"content":"这是一个图片 test\r\nhahaha"}
defaultHandler.handler(m.getData(), responseFunction);
```

native 对 js 进行 callBack，把默认数据 "DefaultHandler response data" 传递给 js，调用 queueMessage(responseMsg);

```
public class DefaultHandler implements BridgeHandler{

	String TAG = "DefaultHandler";

	@Override
	public void handler(String data, CallBackFunction function) {
		if(function != null){
			function.onCallBack("DefaultHandler response data");
		}
	}
}

```

`javascriptCommand`:

```
javascript:WebViewJavascriptBridge._handleMessageFromNative('{\"responseData\":\"DefaultHandler responsedata\",\"responseId\":\"cb_4_1495699731002\"}');
```

queueMessage 中 dispatchMessage(m); 把 native 返回的数据进行处理，使用 webview.loadUrl(javascriptCommand); 把数据传递给 html。

```
private void queueMessage(Message m) {
  if (startupMessage != null) {
    startupMessage.add(m);
  } else {
    dispatchMessage(m);
  }
}

void dispatchMessage(Message m) {
      String messageJson = m.toJson();
      //escape special characters for json string
      messageJson = messageJson.replaceAll("(\\\\)([^utrn])", "\\\\\\\\$1$2");
      messageJson = messageJson.replaceAll("(?<=[^\\\\])(\")", "\\\\\"");
      String javascriptCommand = String.format(BridgeUtil.JS_HANDLE_MESSAGE_FROM_JAVA, messageJson);
      if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
          this.loadUrl(javascriptCommand);
      }
}

```
