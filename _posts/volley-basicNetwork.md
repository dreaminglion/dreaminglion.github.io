---
layout: post
title:  "volley basic network"
date:   2016-11-18 22:15:00 +0800
categories: reader
---

volly 源码阅读，解析 BasicNetwork、HurlStack 篇，执行 request 完成网络请求，得到网络返回数据 NetworkResponse。

<!-- more -->

## BasicNetwork 对 request 做了些什么

由  NetworkResponse networkResponse = mNetwork.performRequest(request); 看到 BasicNetwork 执行 request 得到网络请求结果。其实这一层只是对缓存的 request 做了，处理实际网络请求还是由 HurlStack 执行 request 得到。

```
@Override
   public NetworkResponse performRequest(Request<?> request) throws VolleyError {
       long requestStart = SystemClock.elapsedRealtime();
       while (true) {
           HttpResponse httpResponse = null;
           byte[] responseContents = null;
           Map<String, String> responseHeaders = Collections.emptyMap();
           try {
               // Gather headers.
               Map<String, String> headers = new HashMap<String, String>();
               addCacheHeaders(headers, request.getCacheEntry());
               httpResponse = mHttpStack.performRequest(request, headers);
               StatusLine statusLine = httpResponse.getStatusLine();
               int statusCode = statusLine.getStatusCode();

               responseHeaders = convertHeaders(httpResponse.getAllHeaders());
               // Handle cache validation.
               if (statusCode == HttpStatus.SC_NOT_MODIFIED) {

                   Entry entry = request.getCacheEntry();
                   if (entry == null) {
                       return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null,
                               responseHeaders, true,
                               SystemClock.elapsedRealtime() - requestStart);
                   }

                   // A HTTP 304 response does not have all header fields. We
                   // have to use the header fields from the cache entry plus
                   // the new ones from the response.
                   // http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.3.5
                   entry.responseHeaders.putAll(responseHeaders);
                   return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, entry.data,
                           entry.responseHeaders, true,
                           SystemClock.elapsedRealtime() - requestStart);
               }

               // Some responses such as 204s do not have content.  We must check.
               if (httpResponse.getEntity() != null) {
                 responseContents = entityToBytes(httpResponse.getEntity());
               } else {
                 // Add 0 byte response as a way of honestly representing a
                 // no-content request.
                 responseContents = new byte[0];
               }

               // if the request is slow, log it.
               long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
               logSlowRequests(requestLifetime, request, responseContents, statusLine);

               if (statusCode < 200 || statusCode > 299) {
                   throw new IOException();
               }
               return new NetworkResponse(statusCode, responseContents, responseHeaders, false,
                       SystemClock.elapsedRealtime() - requestStart);
           } catch (SocketTimeoutException e) {
               attemptRetryOnException("socket", request, new TimeoutError());
           } catch (ConnectTimeoutException e) {
               attemptRetryOnException("connection", request, new TimeoutError());
           } catch (MalformedURLException e) {
               throw new RuntimeException("Bad URL " + request.getUrl(), e);
           } catch (IOException e) {
               int statusCode = 0;
               NetworkResponse networkResponse = null;
               if (httpResponse != null) {
                   statusCode = httpResponse.getStatusLine().getStatusCode();
               } else {
                   throw new NoConnectionError(e);
               }
               VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
               if (responseContents != null) {
                   networkResponse = new NetworkResponse(statusCode, responseContents,
                           responseHeaders, false, SystemClock.elapsedRealtime() - requestStart);
                   if (statusCode == HttpStatus.SC_UNAUTHORIZED ||
                           statusCode == HttpStatus.SC_FORBIDDEN) {
                       attemptRetryOnException("auth",
                               request, new AuthFailureError(networkResponse));
                   } else {
                       // TODO: Only throw ServerError for 5xx status codes.
                       throw new ServerError(networkResponse);
                   }
               } else {
                   throw new NetworkError(networkResponse);
               }
           }
       }
}

```

addCacheHeaders(headers, request.getCacheEntry()); 看 request 是否有了缓存，对缓存对应的 header 进行处理。
httpResponse = mHttpStack.performRequest(request, headers); 关键的一步，由 mHttpStack 完成 request 的请求。
得到网络请求结果 httpResponse 的 statusCode ，如果 statusCode == HttpStatus.SC_NOT_MODIFIED 状态没有修改，直接返回 request.getCacheEntry() 作为网络请求的结果。
如果没有缓存，网络请求得到的 httpResponse.getEntity() 服务器给的 null 进行容错  responseContents = new byte[0];
最终 return new NetworkResponse(statusCode, responseContents, responseHeaders, false,SystemClock.elapsedRealtime() - requestStart); 返回网络请求结果。

## HurlStack 真正执行 request 的网络处理

An {@link HttpStack} based on {@link HttpURLConnection}. 看到熟悉的网络工具了吗？ **HttpURLConnection** 书本中通过 HttpURLConnection conn = url.openConnection(); InputStream inStream = conn.getInputStream(); 网络通过 byte[] 文件流读写，传送 request 和 response 的内容。


构造方法，urlRewriter 对 request 的 URL 进行处理的工具，sslSocketFactory 开启 https 的工具。默认使用的构造方法都为 null。

```
/**
 * @param urlRewriter Rewriter to use for request URLs
 * @param sslSocketFactory SSL factory to use for HTTPS connections
 */
public HurlStack(UrlRewriter urlRewriter, SSLSocketFactory sslSocketFactory) {
    mUrlRewriter = urlRewriter;
    mSslSocketFactory = sslSocketFactory;
}

```

执行 request 网络请求，additionalHeaders 缓存对应的额外 header。

```
@Override
public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError {
    String url = request.getUrl();
    HashMap<String, String> map = new HashMap<String, String>();
    map.putAll(request.getHeaders());
    map.putAll(additionalHeaders);
    if (mUrlRewriter != null) {
        String rewritten = mUrlRewriter.rewriteUrl(url);
        if (rewritten == null) {
            throw new IOException("URL blocked by rewriter: " + url);
        }
        url = rewritten;
    }
    URL parsedUrl = new URL(url);
    HttpURLConnection connection = openConnection(parsedUrl, request);
    for (String headerName : map.keySet()) {
        connection.addRequestProperty(headerName, map.get(headerName));
    }
    setConnectionParametersForRequest(connection, request);
    // Initialize HttpResponse with data from the HttpURLConnection.
    ProtocolVersion protocolVersion = new ProtocolVersion("HTTP", 1, 1);
    int responseCode = connection.getResponseCode();
    if (responseCode == -1) {
        // -1 is returned by getResponseCode() if the response code could not be retrieved.
        // Signal to the caller that something was wrong with the connection.
        throw new IOException("Could not retrieve response code from HttpUrlConnection.");
    }
    StatusLine responseStatus = new BasicStatusLine(protocolVersion,
            connection.getResponseCode(), connection.getResponseMessage());
    BasicHttpResponse response = new BasicHttpResponse(responseStatus);
    response.setEntity(entityFromConnection(connection));
    for (Entry<String, List<String>> header : connection.getHeaderFields().entrySet()) {
        if (header.getKey() != null) {
            Header h = new BasicHeader(header.getKey(), header.getValue().get(0));
            response.addHeader(h);
        }
    }
    return response;
}

```

1.创建一个 map 把 request headers 和缓存的 additionalHeaders 加在一起。
2.当 mUrlRewriter 部位空的时候 String rewritten = mUrlRewriter.rewriteUrl(url); 对 url 进行处理。
3.使用 url 打开连接，如果有 ssl 打开 https。

```
private HttpURLConnection openConnection(URL url, Request<?> request) throws IOException {
    HttpURLConnection connection = createConnection(url);

    int timeoutMs = request.getTimeoutMs();
    connection.setConnectTimeout(timeoutMs);
    connection.setReadTimeout(timeoutMs);
    connection.setUseCaches(false);
    connection.setDoInput(true);

    // use caller-provided custom SslSocketFactory, if any, for HTTPS
    if ("https".equals(url.getProtocol()) && mSslSocketFactory != null) {
        ((HttpsURLConnection)connection).setSSLSocketFactory(mSslSocketFactory);
    }

    return connection;
}
```

4.把所有的 headers 添加到 connection 中。

```
for (String headerName : map.keySet()) {
    connection.addRequestProperty(headerName, map.get(headerName));
}
```

5.设置请求方式，并把请求的参数加入 connection 中，setConnectionParametersForRequest(connection, request);。

```
byte[] postBody = request.getPostBody();
if (postBody != null) {
    // Prepare output. There is no need to set Content-Length explicitly,
    // since this is handled by HttpURLConnection using the size of the prepared
    // output stream.
    connection.setDoOutput(true);
    connection.setRequestMethod("POST");
    connection.addRequestProperty(HEADER_CONTENT_TYPE,
            request.getPostBodyContentType());
    DataOutputStream out = new DataOutputStream(connection.getOutputStream());
    out.write(postBody);
    out.close();
}
```

6.设置 http 的协议版本（现在有 http 2 好像很牛） ProtocolVersion protocolVersion = new ProtocolVersion("HTTP", 1, 1);
7.从 connection 中拿到 response 的 statusCode 等信息，创建基础的 BasicHttpResponse

```
StatusLine responseStatus = new BasicStatusLine(protocolVersion,
        connection.getResponseCode(), connection.getResponseMessage());
BasicHttpResponse response = new BasicHttpResponse(responseStatus);

```

8.给 response 设置返回的实体数据，通过输入文件流拿到文件的内容、长度、编码、类型等 response.setEntity(entityFromConnection(connection));

```
private static HttpEntity entityFromConnection(HttpURLConnection connection) {
    BasicHttpEntity entity = new BasicHttpEntity();
    InputStream inputStream;
    try {
        inputStream = connection.getInputStream();
    } catch (IOException ioe) {
        inputStream = connection.getErrorStream();
    }
    entity.setContent(inputStream);
    entity.setContentLength(connection.getContentLength());
    entity.setContentEncoding(connection.getContentEncoding());
    entity.setContentType(connection.getContentType());
    return entity;
}

```

9.把 response 的 header 添加给 response 完成了请求，进行返回。
