---
layout: post
title:  "volley reader"
date:   2016-11-18 22:15:00 +0800
categories: reader
---

volly 源码阅读、解析与实际运用。

<!-- more -->

## volley 核心流程类

1. [volley-request](http://shili.me/2016/11/18/volley-request/) 创建一个请求对象
2. [volley-requestQueue](http://shili.me/2016/11/18/volley-requestQueue/) 创建一个请求的队列
3. [volley-executorDelivery](http://shili.me/2016/11/18/volley-executorDelivery/) 网络请求线程
4. [volley-basicNetwork](http://shili.me/2016/11/18/volley-basicNetwork/) 执行网络请求得到返回结果

## volley 运行过程小结

1.创建一个 request，设置 url、返回的 bean 类型、params 参数、成功和失败回调接口，还有设置 tag 和重试协议。
2.创建一个请求队列，需要 new HurlStack();，new BasicNetwork(stack);，new DiskBasedCache(cacheDir);三个对象，然后调用 queue.start(); 开启队列。
3.在 queue.start() 中 使用缓存队列、网络队列、缓存对象、交付器，创建缓存线程并开启线程，循环创建启动网络线程。
4.现在开启的缓存、网络线程中各个队列还是空的，现在调用 requestQueue 的 add(Request<T> request) 把 request 加入到列队中。
5.然后线程对象们 mDispatchers 开始处理队列中的 request。
6.在网络线程中，网络工具执行请求得到网络返回结果 networkResponse = mNetwork.performRequest(request)。
- BasicNetwork 中对缓存进行了处理，缓存过的 request 拿到缓存数据直接返回，没有缓存的使用 HurlStack 对 request 进行请求。
- HurlStack 使用 HttpURLConnection 实现，完成 request 的发送，response 的返回，以及对 https 的支持，不过对于 http2 需要自己改写 HurlStack 来实现了。
7.把请求到的结果存入缓存集合，然后调用调度器从网络线程发出 mDelivery.postResponse(request, response); mDelivery.postError(request, volleyError);
8.创建请求队列时候，默认的的调度器实现 ExecutorDelivery，调度器默认使用的是主线程的 handler，构造法方法中实例化 Executor handler.post(command); 在主线程中执行 Runnable。
9.在内部类 ResponseDeliveryRunnable 中调用 mRequest.deliverResponse(mResponse.result) 会在主线程执行，request deliverResponse 方法中，成功的 listener 拿到返回的数据更新 UI。
10.在 run 中，接着调用  mRequest.finish("done"); 从队列中删除完成的 request 一个请求完成。

## volley 扩展

1.对于 volley 的扩展用法，在 request 篇已经对本人的一些认识做了说明。
2.还有使用 volley 实现图片上传下载的功能，因项目需求都实现过了，需要的朋友可以邮件我哦。
3.关于 https 公司项目中也使用过了，一种网上很多的说法**允许全部的证书** 就是不对证书验证。另一种只允许自己 app 和服务器配置的证书（这里只做说明，希望大家自己取决使用哪种方式）。
4.对于 http2 我的 leader 测试使用过，速度、稳定都比 http 好很多，http 也是我比较陌生的领域，希望和大家能一起关注学习。

谢谢大家 volley 是自己刚工作就接触的网络库，刚接触的时候感觉没有 [afinal](https://github.com/yangfuhai/afinal) 的 AjaxParams（可以传递各种类型对象，包括文件、流等） 好用。常言说：“贵的东西总是比便宜的好”，从流程程度 volley 显然要高出很多很多。volley 的方便在于开发者，可以实现各种可能的扩展。网络的请求参数最终都是通过 byte[] 二进制字节流的形式发送到服务器，你可以自己实现 volley 的 AjaxParams。好了，就啰嗦到这里，希望和大家一起进行吧！
