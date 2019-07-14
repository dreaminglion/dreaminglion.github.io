---
layout: post
title:  "volley request queue"
date:   2016-11-18 22:15:00 +0800
categories: reader
---

volly 源码阅读，解析 requestQueue 篇，怎么把一个 request 执行，完成请求 -- 加入 requestQueue。

<!-- more -->

## 怎么获取一个 requestQueue

通过一个单例工具，创建管理 mRequestQueue = Volley.newRequestQueue(mCtx.getApplicationContext());
getRequestQueue().add(req); 把 request add 到队列中，volley 框架开始执行这个 request。

并使用 LruBitmapCache 创建 volley 的 ImageLoader，用于加载图片。

```
public class VolleyInstance {

    private static VolleyInstance mInstance;
    private RequestQueue mRequestQueue;
    private ImageLoader mImageLoader;
    private static Context mCtx;

    private VolleyInstance(Context context) {
        mCtx = context;
        mRequestQueue = getRequestQueue();

        mImageLoader = new ImageLoader(mRequestQueue,
                new LruBitmapCache(context));
    }

    public static synchronized VolleyInstance get(Context context) {
        if (mInstance == null) {
            mInstance = new VolleyInstance(context);
        }
        return mInstance;
    }

    public RequestQueue getRequestQueue() {
        if (mRequestQueue == null) {
            // getApplicationContext() is key, it keeps you from leaking the
            // Activity or BroadcastReceiver if someone passes one in.
            mRequestQueue = Volley.newRequestQueue(mCtx.getApplicationContext());
        }
        return mRequestQueue;
    }

    public <T> void addToRequestQueue(Request<T> req) {
        getRequestQueue().add(req);
    }

    public ImageLoader getImageLoader() {
        return mImageLoader;
    }
}

```

## Volley 源码解析

上文使用的 Volley.newRequestQueue(mCtx.getApplicationContext()) 参数 HttpStack 使用的默认 HurlStack。
自己实现 HttpStack 可以使 volley 使用 https 模式，有机会在以后的文章中和大家叨叨。

```
  /** Default on-disk cache directory. */
  private static final String DEFAULT_CACHE_DIR = "volley";

    /**
     * Creates a default instance of the worker pool and calls {@link RequestQueue#start()} on it.
     *
     * @param context A {@link Context} to use for creating the cache dir.
     * @param stack An {@link HttpStack} to use for the network, or null for default.
     * @return A started {@link RequestQueue} instance.
     */
  public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);

        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        queue.start();

        return queue;
  }

```

userAgent 用于 SDK 小于 9 的版本，估计我这辈子也许也没机会用上了啊哈 ~
HurlStack 实现 HttpStack 接口，利用 Java 的 HttpURLConnection 进行各种请求方式的请求。
BasicNetwork 实现 Network，Volley 中默认的网络接口实现类。调用 HttpStack 处理请求，并将结果转换为可被 ResponseDelivery 处理的 NetworkResponse，主要实现了以下功能：
1. 利用 HttpStack 执行网络请求。
2. 如果 Request 中带有实体信息，如 Etag,Last-Modify 等，则进行缓存新鲜度的验证，并处理 304（Not Modify）响应。
3. 如果发生超时，认证失败等错误，进行重试操作，直到成功、抛出异常(不满足重试策略等)结束。

RequestQueue 使用默认的硬盘缓存，和 BasicNetwork 作为参数，创建请求队列，用于将请求 Request 加入到一个运行的 RequestQueue 中，来完成请求操作。

以上部分类描述借助于 [Volley 源码解析](http://a.codekk.com/detail/Android/grumoon/Volley%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)

## RequestQueue 源码解析

### 一个 requestQueue 拥有的一些属性

- private AtomicInteger mSequenceGenerator = new AtomicInteger(); 用于生产 request 的序号。
- private final Map<String, Queue<Request<?>>> mWaitingRequests = new HashMap<String, Queue<Request<?>>>(); 存储 mCacheQueue 中已有的 request。
- **private final Set<Request<?>> mCurrentRequests = new HashSet<Request<?>>(); 正在处理 request 的集合，尚未完成，包含 mCacheQueue、mNetworkQueue 两个队列中的 request。**
- private final PriorityBlockingQueue<Request<?>> mCacheQueue = new PriorityBlockingQueue<Request<?>>(); 优先级阻塞缓存请求队列。
- private final PriorityBlockingQueue<Request<?>> mNetworkQueue = new PriorityBlockingQueue<Request<?>>(); 网络请求队列。
- private static final int DEFAULT_NETWORK_THREAD_POOL_SIZE = 4; 默认的网球请求线程数量。
- private final Cache mCache; 缓存接口对象。
- private final Network mNetwork; 网络请求处理接口对象。
- private final ResponseDelivery mDelivery; 请求返回交付接口对象。
- private NetworkDispatcher[] mDispatchers; 网络请求线程对象数组。
- private CacheDispatcher mCacheDispatcher; 缓存处理线程对象。
- private List<RequestFinishedListener> mFinishedListeners = new ArrayList<RequestFinishedListener>(); 请求完成回调接口数组。

### 一个 requestQueue 拥有的一些方法

构造器方法，需要参数缓存对象，网络对象，网络请求线程数量，请求结果传递器。

```
public RequestQueue(Cache cache, Network network, int threadPoolSize,
        ResponseDelivery delivery) {
    mCache = cache;
    mNetwork = network;
    mDispatchers = new NetworkDispatcher[threadPoolSize];
    mDelivery = delivery;
}

```

请求开发方法，有缓存队列、网络请求队列的线程对象 CacheDispatcher 和 NetworkDispatcher 调用 start() 开启线程处理请求。在 Volley.Java 中 newRequestQueue 的时候调用。

```
public void start() {
     stop();  // Make sure any currently running dispatchers are stopped.
     // Create the cache dispatcher and start it.
     mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
     mCacheDispatcher.start();

     // Create network dispatchers (and corresponding threads) up to the pool size.
     for (int i = 0; i < mDispatchers.length; i++) {
         NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                 mCache, mDelivery);
         mDispatchers[i] = networkDispatcher;
         networkDispatcher.start();
     }
 }

```

停止方法，关闭缓存线程和网络线程。

```
public void stop() {
    if (mCacheDispatcher != null) {
        mCacheDispatcher.quit();
    }
    for (int i = 0; i < mDispatchers.length; i++) {
        if (mDispatchers[i] != null) {
            mDispatchers[i].quit();
        }
}    

```

取消所有以加入到队列的 request，通过给 request 设置的 tag 和 取消参数的 tag 对比，相同的取消请求。

```
public void cancelAll(RequestFilter filter) {
    synchronized (mCurrentRequests) {
        for (Request<?> request : mCurrentRequests) {
            if (filter.apply(request)) {
                request.cancel();
            }
        }
    }
}

/**
 * Cancels all requests in this queue with the given tag. Tag must be non-null
 * and equality is by identity.
 */
public void cancelAll(final Object tag) {
    if (tag == null) {
        throw new IllegalArgumentException("Cannot cancelAll with a null tag");
    }
    cancelAll(new RequestFilter() {
        @Override
        public boolean apply(Request<?> request) {
            return request.getTag() == tag;
        }
    });
}

```

添加 request 方法，同步把每一个 request 添加到 mCurrentRequests 中，不缓存的直接添加到 mNetworkQueue，需要缓存的先加入 mCacheQueue 中，重复的缓存 request 加入 mWaitingRequests 中。

```
public <T> Request<T> add(Request<T> request) {
        // Tag the request as belonging to this queue and add it to the set of current requests.
        request.setRequestQueue(this);
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }

        // Process requests in the order they are added.
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");

        // If the request is uncacheable, skip the cache queue and go straight to the network.
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }

        // Insert request into stage if there's already a request with the same cache key in flight.
        synchronized (mWaitingRequests) {
            String cacheKey = request.getCacheKey();
            if (mWaitingRequests.containsKey(cacheKey)) {
                // There is already a request in flight. Queue up.
                Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
                if (stagedRequests == null) {
                    stagedRequests = new LinkedList<Request<?>>();
                }
                stagedRequests.add(request);
                mWaitingRequests.put(cacheKey, stagedRequests);
                if (VolleyLog.DEBUG) {
                    VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
                }
            } else {
                // Insert 'null' queue for this cacheKey, indicating there is now a request in
                // flight.
                mWaitingRequests.put(cacheKey, null);
                mCacheQueue.add(request);
            }
            return request;
        }
}

```

request 请求完成方法，在 Request.java 的 finish(final String tag) 方法中调用。先从 mCurrentRequests 队列中删除完成的 request，循环执行完成回调接口 listener.onRequestFinished(request)。应当缓存的 request 从等待集合中删除，然后加入缓存队列。

```
<T> void finish(Request<T> request) {
      // Remove from the set of requests currently being processed.
      synchronized (mCurrentRequests) {
          mCurrentRequests.remove(request);
      }
      synchronized (mFinishedListeners) {
        for (RequestFinishedListener<T> listener : mFinishedListeners) {
          listener.onRequestFinished(request);
        }
      }

      if (request.shouldCache()) {
          synchronized (mWaitingRequests) {
              String cacheKey = request.getCacheKey();
              Queue<Request<?>> waitingRequests = mWaitingRequests.remove(cacheKey);
              if (waitingRequests != null) {
                  if (VolleyLog.DEBUG) {
                      VolleyLog.v("Releasing %d waiting requests for cacheKey=%s.",
                              waitingRequests.size(), cacheKey);
                  }
                  // Process all queued up requests. They won't be considered as in flight, but
                  // that's not a problem as the cache has been primed by 'request'.
                  mCacheQueue.addAll(waitingRequests);
              }
          }
      }
  }

```

添加删除完成回调接口。

```
public  <T> void addRequestFinishedListener(RequestFinishedListener<T> listener) {
   synchronized (mFinishedListeners) {
     mFinishedListeners.add(listener);
   }
}

 /**
  * Remove a RequestFinishedListener. Has no effect if listener was not previously added.
  */
 public  <T> void removeRequestFinishedListener(RequestFinishedListener<T> listener) {
   synchronized (mFinishedListeners) {
     mFinishedListeners.remove(listener);
   }
}

```

## NetworkDispatcher 网络线程实现类

在上文 requestQueue 的 start 方法中，开启了线程 NetworkDispatcher 的 start 方法，一个线程就是执行自己的 run 方法。

```
@Override
   public void run() {
       Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
       while (true) {
           long startTimeMs = SystemClock.elapsedRealtime();
           Request<?> request;
           try {
               // Take a request from the queue.
               request = mQueue.take();
           } catch (InterruptedException e) {
               // We may have been interrupted because it was time to quit.
               if (mQuit) {
                   return;
               }
               continue;
           }

           try {
               request.addMarker("network-queue-take");

               // If the request was cancelled already, do not perform the
               // network request.
               if (request.isCanceled()) {
                   request.finish("network-discard-cancelled");
                   continue;
               }

               addTrafficStatsTag(request);

               // Perform the network request.
               NetworkResponse networkResponse = mNetwork.performRequest(request);
               request.addMarker("network-http-complete");

               // If the server returned 304 AND we delivered a response already,
               // we're done -- don't deliver a second identical response.
               if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                   request.finish("not-modified");
                   continue;
               }

               // Parse the response here on the worker thread.
               Response<?> response = request.parseNetworkResponse(networkResponse);
               request.addMarker("network-parse-complete");

               // Write to cache if applicable.
               // TODO: Only update cache metadata instead of entire record for 304s.
               if (request.shouldCache() && response.cacheEntry != null) {
                   mCache.put(request.getCacheKey(), response.cacheEntry);
                   request.addMarker("network-cache-written");
               }

               // Post the response back.
               request.markDelivered();
               mDelivery.postResponse(request, response);
           } catch (VolleyError volleyError) {
               volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
               parseAndDeliverNetworkError(request, volleyError);
           } catch (Exception e) {
               VolleyLog.e(e, "Unhandled exception %s", e.toString());
               VolleyError volleyError = new VolleyError(e);
               volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
               mDelivery.postError(request, volleyError);
           }
       }
}

```

网络线程的 run 方法无限循环的 request = mQueue.take(); 从队列取出 request。
使用网络工具，执行 request 得到网络返回结果 NetworkResponse networkResponse = mNetwork.performRequest(request);
request 解析网络返回，得到返回 response -> Response<?> response = request.parseNetworkResponse(networkResponse);
把符合缓存的 request 得到的 response 数据，存入缓存集合中 mCache.put(request.getCacheKey(), response.cacheEntry);
然后执行传递方法，把 response 从网络线程发送出去 mDelivery.postResponse(request, response);

如果请求出错了：
设置出错用了多少时间 volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
把错误信息从网络线程发送出去 mDelivery.postError(request, volleyError);
