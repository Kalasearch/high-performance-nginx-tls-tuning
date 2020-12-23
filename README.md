# 高性能 Nginx HTTPS 调优

## 原文全文发布于卡拉搜索博客：[高性能 Nginx HTTPS 调优](https://kalasearch.cn/blog/high-performance-nginx-tls-tuning/)

本文全部设置位于 `nginx.conf` 文件中，每个调整选项的说明请[阅读原文](https://kalasearch.cn/blog/high-performance-nginx-tls-tuning/)

### 简介
这篇文章中，我们先介绍 Nginx 中的 TLS 设置有哪些与请求延迟可能相关，如何调整才能最大化加速。然后我们用优化[卡拉搜索](https://kalasearch.cn) Nginx 服务器的实例来分享如何调整 Nginx TLS/SSL 设置，为首次搜索的用户提速 30% 左右。我们会详细讨论每一步我们做了一些什么优化，优化的动机和效果。希望可以对其它遇到类似问题的同学提供帮助。


### 调整项

#### 开启 HTTP/2

HTTP/2 标准是从 Google 的 SPDY 上进行的改进，比起 HTTP 1.1 提升了不少性能，尤其是需要并行多个请求的时候可以显著减少延迟。在现在的网络上，一个网页平均需要请求几十次，而在 HTTP 1.1 时代浏览器能做的就是多开几个连接（通常是 6 个）进行并行请求，而 HTTP 2 中可以在一个连接中进行并行请求。HTTP 2 原生支持多个并行请求，因此大大减少了顺序执行的请求的往返程，可以首要考虑开启。

如果你想自己看一下 HTTP 1.1 和 HTTP 2.0 的速度差异，可以试一下：[https://www.httpvshttps.com/](https://www.httpvshttps.com/)。我的网络测试下来 HTTP/2 比 HTTP 1.1 快了 66%。

![HTTP 1.1 与 HTTP 2.0 速度对比](https://kalasearch.cn/static/147fc37e212cbc9ec6d35a7a8560fa05/29007/HTTP2-speed-compare.png)


在 Nginx 中开启 HTTP 2.0 非常简单，只需要增加一个 http2 标志即可

```jsx
listen 443 ssl;

# 改为
listen 443 ssl http2;
```

如果你担心你的用户用的是旧的客户端，比如 Python 的 requests，暂时还不支持 HTTP 2 的话，那么其实不用担心。如果用户的客户端不支持 HTTP 2，那么连接会自动降级为 HTTP 1.1，保持了后向兼容。因此，所有使用旧 Client 的用户，仍然不受影响，而新的客户端则可以享受 HTTP/2 的新特性。



#### 调整 Cipher 优先级

尽量挑选更新更快的 Cipher，有助于[减少延迟](https://stackoverflow.com/questions/36672261/how-to-reduce-ssl-time-of-website):

```jsx

# 手动启用 cipher 列表
ssl_prefer_server_ciphers on;  # prefer a list of ciphers to prevent old and slow ciphers
ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
```

#### 启用 OCSP Stapling

在国内这可能是对使用 Let's Encrypt 证书的服务或网站影响最大的延迟优化了。如果不启用 OCSP Stapling 的话，在用户连接你的服务器的时候，有时候需要去验证证书。而因为一些不可知的原因（这个就不说穿了）[Let's Encrypt 的验证服务器并不是非常通畅](https://juejin.cn/post/6844904135653851150)，因此可以造成有时候[数秒甚至十几秒延迟的问题](https://jhuo.ca/post/ocsp-stapling-letsencrypt/)，这个问题在 iOS 设备上特别严重

解决这个问题的方法有两个：

1. 不使用 Let's Encrypt，可以尝试替换为阿里云提供的免费 DV 证书
2. 开启 OCSP Stapling

开启了 OCSP Stapling 的话，跑到证书验证这一步可以省略掉。省掉一个 roundtrip，特别是网络状况不可控的 roundtrip，可能可以将你的延迟大大减少。

在 Nginx 中启用 OCSP Stapling 也非常简单，只需要设置：

```jsx
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /path/to/full_chain.pem;
```
#### 调整 `ssl_buffer_size`

ssl_buffer_size 控制在发送数据时的 buffer 大小，默认设置是 16k。这个值越小，则延迟越小，而添加的报头之类会使 overhead 会变大，反之则延迟越大，overhead 越小。


#### 启用 SSL Session 缓存

启用 SSL Session 缓存可以大大减少 TLS 的反复验证，减少 TLS 握手的 roundtrip。虽然 session 缓存会占用一定内存，但是用 1M 的内存就可以缓存 4000 个连接，可以说是非常非常划算的。同时，对于绝大多数网站和服务，要达到 4000 个同时连接本身就需要非常非常大的用户基数，因此可以放心开启。

这里 `ssl_session_cache` 设置为使用 50M 内存，以及 4 小时的连接超时关闭时间 `ssl_session_timeout`

```jsx
# Enable SSL cache to speed up for return visitors
ssl_session_cache   shared:SSL:50m; # speed up first time. 1m ~= 4000 connections
ssl_session_timeout 4h;
```


## 更多文章:

[Grafana 中文教程](https://kalasearch.cn/blog/grafana-with-prometheus-tutorial/)

[REST API 设计指南](https://kalasearch.cn/blog/rest-api-best-practices/)

[ElasticSearch 终极教程 - 第一章](https://kalasearch.cn/blog/elasticsearch-tutorial/)

[ElasticSearch 终极教程 - 第二章](https://kalasearch.cn/blog/chapter2-run-elastic-search-locally/)

[ElasticSearch 终极教程 - 第三章](https://kalasearch.cn/blog/chapter3-elastic-search-and-lucene/)


