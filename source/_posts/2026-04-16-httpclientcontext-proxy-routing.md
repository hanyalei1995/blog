---
title: Spring Boot 代理路由排障：为什么 HttpClientContext 没有走代理
date: 2026-04-16 14:35:00
tags:
  - Java
  - Spring Boot
  - HttpClient
  - Feign
  - 代理
categories:
  - 后端开发
description: 结合一个 Spring Boot 代理 starter 的实现，分析 HttpClientContext 调用没有走代理的根因，并给出最小修复方案。
top_img:
cover:
password:
message:
---

最近在排查一个代理问题时，表面现象很奇怪：同一套 `igs.proxy` 配置下，Feign 调外部平台接口可以正常走代理，但 `HttpClientContext` 发出去的请求却直接连目标地址，没有经过代理服务器。

这类问题如果只盯着配置，很容易一直绕在 `pattern`、`match_url`、`feign-packages` 这些字段上。真正把代码链路走一遍之后，根因其实很清楚。

本文就结合一个 Spring Boot 代理 starter 模块，把这套代理实现逻辑和最终修复方案完整梳理一下。

## 先看配置

当时的配置大概是这样：

```yaml
igs:
  proxy:
    enable: true
    feign-packages:
      - com.example.bridge.external.api.*
    servers:
      - host: proxy-host-1
        port: 10080
        mode: match_url
        patterns:
          - https://openapi-b-us.example.com/**
      - host: proxy-host-1
        port: 10080
        mode: match_url
        patterns:
          - https://openapi-b-eu.example.com/**
```

实际访问的 URL 是一个外部平台的文件下载地址：

```text
https://openapi-b-us.example.com/file-download/***.pdf?sign=***
```

先说结论：这个 URL 和上面的 `pattern` 是能匹配上的，配置本身没有问题。

## 这套代理实现是怎么工作的

整个模块里，代理能力主要分成三层：

### 1. 自动装配层

`ProxyAutoConfiguration` 在 `igs.proxy.enable=true` 时生效，创建一个自定义的 `ProxyHttpClient`，然后做两件事：

- 给 `RestTemplate` 使用
- 通过 `HttpClientContext.bindIgsProxyHttpClient(...)` 绑定到静态工具类

同时，它还会创建 `ProxyAwareTargeter`，让符合包名规则的 Feign 接口走代理客户端。

### 2. Feign 路由层

`ProxyAwareTargeter` 的作用不是“匹配 URL 是否走代理”，而是“决定这个 Feign 客户端到底使用哪个 Client”。

它的判断规则是：

- 命中 `igs.proxy.feign-packages` 的 Feign 接口，使用代理 `HttpClient`
- 没命中的内部服务调用，继续使用 Spring Cloud 默认的负载均衡 Client

这里有一个非常容易混淆的点：

`feign-packages` 只影响 Feign。

它对 `HttpClientContext` 没有任何作用。

### 3. URL 匹配层

真正决定某个请求要不要走代理的，是 `ProxyRoutePlanner` 和 `ProxyUrlMatcher`。

执行逻辑大概是这样：

1. `ProxyHttpClient.doExecute()` 拿到当前请求
2. 根据请求 URL 和 `igs.proxy.servers[*].patterns` 找到匹配的代理节点
3. 把匹配到的代理信息放进 `HttpContext`
4. `ProxyRoutePlanner.determineRoute()` 再决定最终路由是“经代理”还是“直连”

如果能匹配到代理节点，就会构造这样的路由：

```java
return new HttpRoute(target, null, proxyHost, secure);
```

如果没有匹配到，就直接：

```java
return new HttpRoute(target);
```

## 为什么 Feign 正常，HttpClientContext 却不走代理

问题就出在 `HttpClientContext` 这条链路上。

先看 `HttpClientContext` 的发送逻辑，它最终是这样的：

```java
response = httpClient.execute(requestBase);
```

也就是说，它底层确实是走了已经绑定进去的 `ProxyHttpClient`，理论上应该能复用同一套路由能力。

但继续往下跟，会发现有个关键细节。

### doExecute 阶段已经匹配过一次代理

在 `ProxyHttpClient.doExecute()` 里，请求执行前会先做一次匹配：

```java
ProxyProperties.ProxyServer matchedProxy =
        urlMatcher.getMatchedProxy(httpRequest, proxyProperties.getServers());
```

如果找到了代理节点，就会把它放进上下文：

```java
context.setAttribute("selectedProxy", matchedProxy);
```

按理说，这时候代理已经选好了。

### RoutePlanner 阶段又重新匹配了一次

旧实现里的 `ProxyRoutePlanner.determineRoute()` 没有优先使用这个已经选好的 `selectedProxy`，而是重新拿 `request` 去匹配：

```java
ProxyProperties.ProxyServer matchedProxy =
        urlMatcher.getMatchedProxy(request, proxyProperties.getServers());
```

这一步就是问题所在。

因为 Apache HttpClient 到了路由阶段，`request` 不一定还是最初那个完整绝对 URL。很多时候它只剩下相对路径，比如：

```text
/download-path/***.pdf?signature=***
```

而我们的配置模式是：

```text
https://openapi-b-us.example.com/**
```

这时重新匹配当然就失败了，最终路由退化成直连。

## 根因总结

不是配置错了，也不是 `HttpClientContext` 完全没用代理客户端。

真正的根因是：

- `ProxyHttpClient` 在执行前已经匹配到了代理
- 但 `ProxyRoutePlanner` 在路由阶段没有复用这个结果
- 而是对可能已经变成“相对 URI”的请求重新匹配
- 最终导致匹配丢失，请求走直连

这也是为什么 Feign 看起来是正常的，而 `HttpClientContext` 暴露出了问题。

## 最小修复方案

最稳妥的改法不是重写匹配规则，而是让 `ProxyRoutePlanner` 优先使用 `HttpContext` 中已经选好的代理：

```java
ProxyProperties.ProxyServer matchedProxy = getSelectedProxy(context);
if (matchedProxy == null) {
    matchedProxy = urlMatcher.getMatchedProxy(request, proxyProperties.getServers());
}
```

对应的辅助方法也很简单：

```java
private ProxyProperties.ProxyServer getSelectedProxy(HttpContext context) {
    if (context == null) {
        return null;
    }
    Object selectedProxy = context.getAttribute("selectedProxy");
    if (selectedProxy instanceof ProxyProperties.ProxyServer) {
        return (ProxyProperties.ProxyServer) selectedProxy;
    }
    return null;
}
```

这样改完之后，执行链路就变成了：

1. `ProxyHttpClient` 先根据完整 URL 选出代理
2. 把代理节点写入 `HttpContext`
3. `ProxyRoutePlanner` 优先读取上下文中的 `selectedProxy`
4. 只有上下文没有值时，才回退到重新匹配

这个改动非常小，但能把整个行为修正回来，而且不会影响 Feign 的现有逻辑。

## 回归测试怎么写

为了避免以后再出现同样的问题，我补了一个针对性的单测，场景就是：

- `HttpContext` 里已经放入了选中的代理
- 传给 `RoutePlanner` 的请求只剩相对路径
- 最终仍然应该返回代理路由

测试核心断言如下：

```java
assertEquals("proxy-host-1", route.getProxyHost().getHostName());
assertEquals(10080, route.getProxyHost().getPort());
```

在 Java 1.8 环境下跑完整个代理 starter 模块测试，结果是：

- `Tests run: 13`
- `Failures: 0`
- `Errors: 0`

说明这次修复既覆盖了目标问题，也没有破坏原有的 Feign 包路由逻辑。

## 这个问题给我的两个提醒

### 1. 不要把“Feign 是否走代理”和“URL 是否匹配代理”混为一谈

这是两层不同的判断：

- `feign-packages` 决定 Feign 用哪个 `Client`
- `servers.patterns` 决定具体请求是否匹配代理节点

很多排障时间都浪费在把这两个维度混在一起看。

### 2. 上下文里已经算出的结果，后续阶段尽量复用

如果一个请求在前置阶段已经完成了代理选择，后面最好直接复用，不要依赖“再次推导”。

因为请求对象在不同执行阶段很可能发生变化：

- 绝对 URL 变相对路径
- host 信息从 request 挪到 target
- query string 表达方式变化

这些都会让“重新匹配”变得不可靠。

## 总结

这次问题的表象是 `HttpClientContext` 没有走代理，但真正原因并不是配置错误，而是路由阶段把已经匹配成功的代理信息弄丢了。

修复思路也很简单：

- 不改现有配置协议
- 不改 Feign 的包路由策略
- 只让 `ProxyRoutePlanner` 优先读取 `HttpContext` 里的 `selectedProxy`

这种修法改动小、风险低，而且逻辑上也更合理。

如果你在项目里也遇到“同样配置下，有的调用走代理、有的调用不走代理”的情况，建议第一时间把整条调用链分层看清楚：到底是客户端选择错了，还是路由阶段把已经选好的代理丢了。
