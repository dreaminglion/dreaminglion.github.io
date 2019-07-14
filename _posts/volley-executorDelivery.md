---
layout: post
title:  "volley executor delivery"
date:   2016-11-18 22:15:00 +0800
categories: reader
---

ExecutorDelivery 传递器 volley 包中实现的类，用于传递网络请求成功、失败的结果。  

<!-- more -->

## ExecutorDelivery

在 RequestQueue 的默认构造方法中实例化，默认参数是主线程的 handler。

```
public RequestQueue(Cache cache, Network network, int threadPoolSize) {
      this(cache, network, threadPoolSize,
              new ExecutorDelivery(new Handler(Looper.getMainLooper())));
}

```

构造方法中实现一个 Executor 用于在 handler 中的线程中执行 Runnable 的 run 方法代码块。

```
public ExecutorDelivery(final Handler handler) {
      // Make an Executor that just wraps the handler.
      mResponsePoster = new Executor() {
          @Override
          public void execute(Runnable command) {
              handler.post(command);
          }
      };
}

```

调用这个方法，调用内部类实现的 Runable。

```
@Override
public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
      request.markDelivered();
      request.addMarker("post-response");
      mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
}

```

ResponseDeliveryRunnable 的 run 方法会在 handler 的线程中执行，默认使用的是主线程。
mRequest.deliverResponse(mResponse.result); 调用 request 的传递方法，把请求返回结果当参数传入，成功的回调接口使用返回结果进行 UI 的处理显示。
mRequest.finish("done"); 然后调用 finish 从队列中删除这个完成的 request 一个请求完成。

```
private class ResponseDeliveryRunnable implements Runnable {
       private final Request mRequest;
       private final Response mResponse;
       private final Runnable mRunnable;

       public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
           mRequest = request;
           mResponse = response;
           mRunnable = runnable;
       }

       @SuppressWarnings("unchecked")
       @Override
       public void run() {
           // If this request has canceled, finish it and don't deliver.
           if (mRequest.isCanceled()) {
               mRequest.finish("canceled-at-delivery");
               return;
           }

           // Deliver a normal response or error, depending.
           if (mResponse.isSuccess()) {
               mRequest.deliverResponse(mResponse.result);
           } else {
               mRequest.deliverError(mResponse.error);
           }

           // If this is an intermediate response, add a marker, otherwise we're done
           // and the request can be finished.
           if (mResponse.intermediate) {
               mRequest.addMarker("intermediate-response");
           } else {
               mRequest.finish("done");
           }

           // If we have been provided a post-delivery runnable, run it.
           if (mRunnable != null) {
               mRunnable.run();
           }
      }
}

```
