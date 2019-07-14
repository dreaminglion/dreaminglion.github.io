---
layout: post
title:  "volley request"
date:   2016-11-18 22:15:00 +0800
categories: reader
---

volly 源码阅读，解析 request 篇。

<!-- more -->

## 一个 request 我们需要做些什么

常见的两种 request 提交方式 1：Content-Type application/x-www-form-urlencoded 2：Content-Type application/json

### form 表单 request 的使用

在父类 request 中，已默认实现 getBodyContentType 为 application/x-www-form-urlencoded 表单请求。

```
public String getBodyContentType() {
    return "application/x-www-form-urlencoded; charset=" + getParamsEncoding();
}

```

创建一个 request 请求对象

```
/**
 * 创建一个 gson 的请求对象
 *
 * @param method  请求的方式 get post delete put 等
 * @param url     请求的路径 url
 * @param clazz   请求成功，返回的数据类型
 * @param headers 请求的 headers
 * @param params  请求的参数
 * @param listener  请求成功的回调
 * @param errorListener  请求失败的回调
 */
public GsonRequest(int method,String url, Class<T> clazz, Map<String, String> headers, Map<String, String> params,
                   Response.Listener<T> listener, Response.ErrorListener errorListener) {
    super(method, url, errorListener);
    this.clazz = clazz;
    this.headers = headers;
    this.params = params;
    this.listener = listener;
}
```

实现 request 必要的传递 headers、params 和 abstract 方法

```
@Override
public Map<String, String> getHeaders() throws AuthFailureError {
    return headers != null ? headers : super.getHeaders();
}

@Override
protected Map<String, String> getParams() throws AuthFailureError {
    return params;
}

```

解析网络请求返回的 response ，获取 response.data 构建 gson.fromJson(json, clazz) 对象，和网络返回的 headers HttpHeaderParser.parseCacheHeaders(response)。
- 扩展：业务 code 与 volley status code 共同回调 errorListener。

```
@Override
protected Response<T> parseNetworkResponse(NetworkResponse response) {
       try {

           String json = new String(
                   response.data,
                   HttpHeaderParser.parseCharset(response.headers));

                   // BaseBean 为传入类型 clazz 的父类
                   BaseBean bean = Response.success(
                           gson.fromJson(json, clazz),
                           HttpHeaderParser.parseCacheHeaders(response));

          if (bean.code == success_code) {
             return bean;
          } else {
            return Response.error(new ParseError(response));
          }                 
       } catch (UnsupportedEncodingException e) {
           return Response.error(new ParseError(e));
       } catch (JsonSyntaxException e) {
           return Response.error(new ParseError(e));
       }
}
```

执行请求成功的回调，完成一次网络请求。

```
@Override
protected void deliverResponse(T response) {
    listener.onResponse(response);
}
```

### gson 体 request 的使用

在父类 request 中，已默认实现 getBodyContentType 为 application/x-www-form-urlencoded 表单请求。所以实现 gson 请求我们需要复写父类的 getBodyContentType 方法，其它的可与上文保持一致。

```
/**
  * Default charset for JSON request.
  */
protected static final String PROTOCOL_CHARSET = "utf-8";

  /**
   * Content type for request.
   */
private static final String PROTOCOL_CONTENT_TYPE =
        String.format("application/json; charset=%s", PROTOCOL_CHARSET);

@Override
public String getBodyContentType() {
        return PROTOCOL_CONTENT_TYPE;
}        
```

创建一个 gson 请求体的 request 对象，我们也可以不使用 map params 的方式传递参数，由 requestBody 代替，传递 request 的请求参数，这样可以自由的传递参数。会使 getParams 方法返回的参数，不会再代码中使用到，下文源码解析中有说到。

```
/**
 * 创建一个 gson 的请求对象
 *
 * @param method  请求的方式 get post delete put 等
 * @param url     请求的路径 url
 * @param clazz   请求成功，返回的数据类型
 * @param headers 请求的 headers
 * @param requestBody  请求的参数
 * @param listener  请求成功的回调
 * @param errorListener  请求失败的回调
 */
public GsonRequest(int method,String url, Class<T> clazz, Map<String, String> headers, String requestBody,
                   Response.Listener<T> listener, Response.ErrorListener errorListener) {
    super(method, url, errorListener);
    this.clazz = clazz;
    this.headers = headers;
    this.requestBody = requestBody;
    this.listener = listener;
}

```

传递 headers、解析 response 、分发请求与上相同。传递请求体 getParams() 由 getBody() 代替。

```
@Override
public byte[] getBody() {
      try {
          return mRequestBody == null ? null : mRequestBody.getBytes(PROTOCOL_CHARSET);
      } catch (UnsupportedEncodingException uee) {
          VolleyLog.wtf("Unsupported Encoding while trying to get the bytes of %s using %s", mRequestBody, PROTOCOL_CHARSET);
          return null;
      }
}

```

## Request<T> 源码分析

### 一个 request 该有的一些属性

- private final int mMethod; 请求方法
- private final String mUrl; 请求路径
- private final Response.ErrorListener mErrorListener; 失败的 listener
- private final int mDefaultTrafficStatsTag; url 的 host.hashCode()，Default tag for {@link TrafficStats} 不知道干什么用滴 55 ~
- private Integer mSequence; 请求的序列号，用于 FIFO 先进先出
- private RequestQueue mRequestQueue; 请求所在的队列，用户完成请求从该队列删除
- private boolean mShouldCache = true; 是否进行缓存
- private boolean mCanceled = false; 是否取消了本次 request
- private boolean mResponseDelivered = false; 返回的结果 response 是否被传递下去
- private long mRequestBirthTime = 0; 请求开始的时间，用于计算一个请求用时
- private RetryPolicy mRetryPolicy; 请求的重试协议，隔多少秒、重试多少次
- private Cache.Entry mCacheEntry = null; request 对应的缓存
- private Object mTag; 一个 request 的标签，可在请求队列使用这个标签，取消标签对应的所有 request

### 一个 request 该有的一些方法

设置请求的 tag

```
public Request<?> setTag(Object tag) {
        mTag = tag;
        return this;
}
```

设置 retry 协议

```
public Request<?> setRetryPolicy(RetryPolicy retryPolicy) {
       mRetryPolicy = retryPolicy;
       return this;
}
```

完成请求，在队列中删除本次 request

```
void finish(final String tag) {
        if (mRequestQueue != null) {
            mRequestQueue.finish(this);
        }
}
```

设置 request 的请求队列

```
public Request<?> setRequestQueue(RequestQueue requestQueue) {
       mRequestQueue = requestQueue;
       return this;
}
```

设置 request 的序列号

```
public final Request<?> setSequence(int sequence) {
       mSequence = sequence;
       return this;
}

```

设置 request 的缓存

```
public Request<?> setCacheEntry(Cache.Entry entry) {
        mCacheEntry = entry;
        return this;
}

```

取消 request 请求

```
public void cancel() {
        mCanceled = true;
}

```

获取 request 的请求 headers，父类返回一个空的 map，一般子类去具体实现。

```
public Map<String, String> getHeaders() throws AuthFailureError {
        return Collections.emptyMap();
}

```

获取 request 的 params，父类的返回 null

```
protected Map<String, String> getParams() throws AuthFailureError {
       return null;
}

```

获取 Conten-Type、Encoding 父类有默认的实现，默认的 form，UTF-8

```
protected String getParamsEncoding() {
      return DEFAULT_PARAMS_ENCODING;
}

  /**
   * Returns the content type of the POST or PUT body.
   */
public String getBodyContentType() {
      return "application/x-www-form-urlencoded; charset=" + getParamsEncoding();
}

```

设置是否需要缓存

```
public final Request<?> setShouldCache(boolean shouldCache) {
        mShouldCache = shouldCache;
        return this;
}

```

设置 request 的优先级

```
public Priority getPriority() {
        return Priority.NORMAL;
}

```

两个子类必须实现的 abstract 解析成功方法、分发成功的 response

```
abstract protected Response<T> parseNetworkResponse(NetworkResponse response);

abstract protected void deliverResponse(T response);

```

解析失败的 response 由父类实现

```
protected VolleyError parseNetworkError(VolleyError volleyError) {
     return volleyError;
}

 /**
  * Delivers error message to the ErrorListener that the Request was
  * initialized with.
  *
  * @param error Error details
  */
 public void deliverError(VolleyError error) {
     if (mErrorListener != null) {
         mErrorListener.onErrorResponse(error);
     }
 }

```

比较其它 request 的优先级，对 request 进行排序，先进先出，进行请求。

```
@Override
public int compareTo(Request<T> other) {
       Priority left = this.getPriority();
       Priority right = other.getPriority();

       // High-priority requests are "lesser" so they are sorted to the front.
       // Equal priorities are sorted by sequence number to provide FIFO ordering.
       return left == right ?
               this.mSequence - other.mSequence :
               right.ordinal() - left.ordinal();
}

```

**其实 form 表达传递的 params 最后也会被解析为一个 byte[] 作为请求的 body，复写这个方法，可以实现 volley 传递复杂的参数，类似 afinal 中的 AjaxParams**

```
public byte[] getBody() throws AuthFailureError {
       Map<String, String> params = getParams();
       if (params != null && params.size() > 0) {
           return encodeParameters(params, getParamsEncoding());
       }
       return null;
}

   /**
    * Converts <code>params</code> into an application/x-www-form-urlencoded encoded string.
    */
private byte[] encodeParameters(Map<String, String> params, String paramsEncoding) {
       StringBuilder encodedParams = new StringBuilder();
       try {
           for (Map.Entry<String, String> entry : params.entrySet()) {
               encodedParams.append(URLEncoder.encode(entry.getKey(), paramsEncoding));
               encodedParams.append('=');
               encodedParams.append(URLEncoder.encode(entry.getValue(), paramsEncoding));
               encodedParams.append('&');
           }
           return encodedParams.toString().getBytes(paramsEncoding);
       } catch (UnsupportedEncodingException uee) {
           throw new RuntimeException("Encoding not supported: " + paramsEncoding, uee);
       }
}

```

## volley 用法扩展

问题：volley 的 params 使用的是 map 进行传递的，许多网络框架 params 使用的是 list 传递，list 可以传递相同的 key，对传递数组比较方便，而 map 一般同学会认为 key-value 会覆盖上次的 value 无法传递数组。

1.使用 IdentityHashMap 这个 map 的 key 使用的是 key 的内存地址，并不是 key 所对应的值。如下通过 new String("d") 作为 key 使用 volley 的 getParams() 就可以把 list 这个数组作为参数。

```
IdentityHashMap identityHashMap = new IdentityHashMap();
identityHashMap.put("m", formData.m);
identityHashMap.put("t", System.currentTimeMillis() + "");
for (int i = 0; i < list.size(); i++) {
    identityHashMap.put(new String("d"), list.get(i));
}
```

2.在自定义的 GsonRequest 中复写 byte[] getBody() 方法，自己对参数 params 进行封装，使用上文粗体方法 StringBuilder encodedParams = new StringBuilder() 对 params 进行拼接，并返回拼接后的 byte[] return encodedParams.toString().getBytes(paramsEncoding);
