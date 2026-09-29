# Java 服务的出网代理与请求头透传边界

> 场景：一个常驻 Java 服务既要访问公网第三方地址（有时必须经代理出网），又在同进程里跑着大量内部 RPC 调用与第三方 SDK。
> 「代理的作用范围」和「鉴权头发给谁」这两件事如果没有边界意识，很容易变成：改一个开关，整个服务的 HTTP 都跟着跑偏。
> 本文只讲通用原理与判断依据，示例均为自行构造的中立写法。

## 1. 配代理有三种作用域，优先级从高到低

| 方式 | 作用范围 | 主要风险 |
|------|---------|---------|
| 连接级：`URL.openConnection(Proxy)`、`Socket` 连接指定 `Proxy` | 仅这一次连接 | 无副作用，只是要自己传参 |
| 客户端级：HttpClient `setProxy`、OkHttp `proxy()`、RestTemplate/WebClient 的请求器配置 | 该客户端实例 | 需要为不同出口准备不同实例 |
| JVM 全局：`-Dhttp.proxyHost` / `-Dhttps.proxyHost` / `-DsocksProxyHost`、`System.setProperty` | 进程内所有 `URLConnection` | 内部调用、SDK、健康检查全被牵走 |

结论：**能在连接级或客户端级解决的问题，绝不动 JVM 全局属性。**

几个容易踩的细节：

1. `http.proxyHost` 与 `https.proxyHost` 是**两个独立开关**，只配一个就会出现"HTTP 请求走代理、HTTPS 请求直连"的怪现象；
2. `http.nonProxyHosts` 白名单只对 JDK `URLConnection` 体系生效，Apache HttpClient / OkHttp 根本不认它，用它兜内网地址一定会漏；
3. `-Djava.net.useSystemProxies=true` 会去读操作系统级代理配置，行为随运行环境漂移，服务端不建议依赖；
4. `ProxySelector.getDefault()` 看似方便，但同样会把环境差异带进程序；对"可开可关的代理"这种配置，更推荐显式传参、传 null 即直连。

## 2. JDK 连接级代理的最小写法

```java
// host 留空或 port 非法时返回 null，表示直连 —— 开关留在配置里，不必改代码或重启加 JVM 参数
Proxy proxy = (host == null || host.isBlank() || port <= 0)
        ? null
        : new Proxy(Proxy.Type.HTTP, new InetSocketAddress(host, port));

// URI.create(...).toURL() 是现代写法，new URL(String) 在 JDK 20 起已标记待废弃
URLConnection connection = URI.create(url).toURL().openConnection(proxy);
```

要点：

1. `Proxy.Type.HTTP` 与 `Proxy.Type.SOCKS` 不能混用，把 SOCKS 端口当 HTTP 代理配置会直接连不上；
2. **HTTPS 目标走 HTTP 代理时，JDK 会自动发起 `CONNECT` 建立隧道**，业务代码不需要任何额外处理；代理侧若禁止 `CONNECT`，表现为连接阶段就失败而不是证书错误；
3. 代理需要认证时另配 `java.net.Authenticator`，或换用支持代理认证的客户端；
4. 重定向（302）默认行为在跨协议/跨主机时有限制，需要跳转就用能显式控制跳转的策略，别默认"肯定能跟"。

## 3. 下载类请求的四个必做检查

1. **超时拆成两段**。连接超时（含与代理建链）和读超时（等数据）语义完全不同。大文件读超时要给足，给短了会表现为"下载到快结束时失败"，比一开始就失败更难判断原因。
2. **状态码先行**。非 2xx 直接抛异常，并把状态码与 URL 写进异常信息。若把 404/403 的响应体当文件存下去，会得到一个"扩展名正常、内容却是 HTML"的坏文件，故障延后到读取时才炸，链路看起来毫无关联。
3. **字节数组还是流**。小文件读进内存最省事；大文件必须改成 `InputStream` → 目标 `OutputStream`（或直接交给对象存储 SDK）流式转发，否则单个请求就吃掉几十 MB 堆内存，并发下很容易 OOM。能拿到 `Content-Length` 时读完做一次长度比对，可以识别"中途截断却当成功"的情况。
4. **一定释放连接**。`finally` 中关闭流并 `disconnect()`，异常路径同样要释放，否则句柄与连接泄漏。

日志建议：把"状态码 + 本次耗时 + 走的通道（代理地址或直连）+ URL"打在一行，把"字节数 + 总耗时"打在另一行。这三类信息齐了，绝大多数出网问题不用抓包就能定位。

## 4. 重试策略属于调用方，不属于工具方法

工具方法只保证"发一次请求、如实报告失败"，重试次数、间隔、退避、失败后怎么办由业务决定 —— 同一个下载动作，在 MQ 消费链路和在人工运维脚本里，该不该重试的结论往往相反。

- **区分可重试与不可重试**：连接超时、读超时、5xx、429 一般值得重试；404、403、重复的 3xx 再试也没有意义；
- 固定间隔适合低频运维任务，在线链路建议**指数退避 + 抖动**，并给总耗时设上限（而不是只限次数）；
- 捕获 `InterruptedException` 后必须 `Thread.currentThread().interrupt()` 恢复中断标志再抛出，否则上层线程池的取消与关停会失效；
- 最终异常保留**最后一次原始异常**作为 cause，丢掉它等于丢掉真正的原因；
- 重试日志要能区分三种现象：连不上代理 / 代理通了但目标返回非 2xx / 传输中途断开。

## 5. 鉴权请求头的透传边界

常见诉求：服务 A 代表登录用户调用服务 B，B 需要知道"操作人是谁"，于是把入口请求的 `Authorization` 透传到下游。三条通用注意事项：

1. **只透给内部目标**。写在全局的 Feign / HTTP 拦截器很容易把内部令牌顺带发给外部域名（支付、短信、AI 接口）。
   一个可用的判据是目标主机形态：服务发现目标的 host 是服务名（通常不含点），硬编码的外部域名与 IP 都含点；
   但这只是兜底判断，**更稳的做法是显式白名单，或给外部调用单独配置一套不带透传的客户端**。
2. **异步线程、定时任务、消息消费里没有请求上下文**，取到 null 是正常情况，应当静默跳过而不是抛 NPE —— 这些链路本来就不存在"当前登录用户"。
3. **下游要把"取不到操作人"当业务错误处理**。`Objects.requireNonNull(...)` 这类写法会把它变成一条 NPE，既不可读，又掩盖了"上游没把登录态传过来"这一真实原因；正确做法是显式判空后返回可理解的提示，或按业务约定写入明确的系统账号。

另外：透传出去的凭据会被下游打进日志的风险普遍存在，**请求头日志脱敏**（`Authorization`、`Cookie`）要和透传一起考虑，否则等于扩大了凭据暴露面。

## 6. 客户端断开连接不算业务失败

前端取消请求（页面跳转、重复点击 abort、上游网关超时）会让服务端在**写响应**阶段抛异常：Servlet 栈下常见 Tomcat `ClientAbortException`，异步/响应式返回下是 Spring 的 `AsyncRequestNotUsableException`。此时业务逻辑通常已经执行完毕，只是响应送不出去。

- 处理分支返回 `void`，不要再尝试写错误响应体（连接已断，写了必然再失败一次）；
- 只记一行 warn（请求方法、路径、traceId），**不打完整堆栈**，否则日志刷屏会把真实异常淹没；
- 排查顺序：先在 warn 级别找"客户端已断开"，再用 traceId 串起业务日志确认业务本身是否执行成功；
- 对"已执行成功但响应未送达"的写操作，只有幂等设计或先落状态再返回才算安全 —— 用户看不到结果就会重试。

## 7. 快速验证手段

1. 先用 curl 单独确认代理可用与目标可达，再回到代码：
   `curl -x http://<proxy-host>:<port> -o /dev/null -w "%{http_code} %{time_total}\n" <url>`
2. 去掉 `-x` 直连跑一次对比耗时与成功率，确认"代理是否真的解决问题"，避免为一个不存在的问题长期挂代理；
3. `curl -v` 的输出可直接判断是否走代理：显示 `Connected to <代理IP>` 即走了代理；访问 HTTPS 目标时代理侧应出现 `HTTP/1.1 200 Connection established`（CONNECT 隧道建好）；
4. 服务侧做一次破坏性验证：把代理端口临时改成不存在的端口，预期是"按设定的次数重试后失败并抛出最后一次原因"，而不是静默成功或一直挂住；
5. 透传验证：打开外部调用的请求日志，确认外部请求头里**没有** `Authorization` 与 `Cookie`。

## 8. 一句话原则

- 代理作用域能小就不大：连接级 > 客户端级 > JVM 全局；
- 工具方法不做重试，只把失败信息说清楚；
- 状态码先判、长度再校验、连接必释放；
- 内部凭据不出内网，透传前先确认目标是谁；
- 客户端断连记 warn，业务失败才记 error。

## 9. 参考资料

查阅官方文档后自行整理表述，未复制正文：

- JDK `java.net.Proxy`、`URL.openConnection(Proxy)`：<https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/net/Proxy.html>
- MDN Web Docs — HTTP response status codes：<https://developer.mozilla.org/en-US/docs/Web/HTTP/Status>
- Spring Cloud OpenFeign — `RequestInterceptor` 与全局配置：<https://spring.github.io/spring-cloud-openfeign/>
- Spring Framework — `AsyncRequestNotUsableException`（Javadoc）：<https://docs.spring.io/spring-framework/>
