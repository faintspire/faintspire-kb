# XXL-JOB 执行器对接文档

| 项目 | 内容 |
|---|---|
| 适用对象 | 所有接入统一调度中心的 Spring Boot 项目（JDK 17+ / Spring Boot 3.x） |
| 示例项目 | biovisart-server（Spring Boot 3.4.13 / JDK 21） |
| 调度中心地址 | `http://<调度中心宿主IP>:8081/xxl-job-admin`（见《XXL-JOB部署文档》） |
| 更新日 | 2026-09-17 |

> 📌 **可公开版本说明**：本文档中的 `accessToken` 等敏感值**已脱敏**为占位符；真实令牌为平台级共享秘密，向运维索取或见《XXL-JOB部署文档》compose / Nacos 秘密项，**不得**写入任何公开页面、聊天或截图。

---

## 1. 接入前确认

| 检查项 | 要求 |
|---|---|
| JDK | 17+（xxl-job 3.x 执行器要求；21 兼容） |
| Spring Boot | 3.x（3.4.x 已验证） |
| 网络 | 本机能访问 admin 8081；admin 容器能回访本机执行器端口（biovisart 为 11011） |
| AppName | 与运维约定好「项目名-环境」，如 `biovisart-server-dev` |
| accessToken | 平台级共享令牌，向运维索取（当前值见部署文档 compose，迁移 Nacos 后统一从秘密项读取） |

---

## 2. 引入依赖

Maven 坐标（版本与调度中心保持同线）：

```xml
<dependency>
    <groupId>com.xuxueli</groupId>
    <artifactId>xxl-job-core</artifactId>
    <version>3.4.2</version>
</dependency>
```

多模块项目建议：根 pom `<properties>` 加 `<xxl-job.version>3.4.2</xxl-job.version>` 并在 `dependencyManagement` 收敛版本，业务模块（如 biovisart-biz）只声明 groupId/artifactId。

---

## 3. 配置

### 3.1 应用配置（当前阶段：写死）

```yaml
xxl:
  job:
    admin:
      addresses: http://<调度中心宿主IP>:8081/xxl-job-admin
    accessToken: <平台共享令牌-已脱敏>   # 向运维索取；迁移 Nacos 后替换为占位符
    executor:
      appname: 项目名-@profile.active@      # 如 biovisart-server-@profile.active@ → biovisart-server-dev / -prod
      port: 11011                # 执行器回调端口，各项目自定义（biovisart 统一 11011），不得与业务端口冲突
      logpath: /app/logs/xxl-job
      logretentiondays: 30
```

要点：

- `appname` 必须与 admin「执行器管理」中登记的 AppName **完全一致**，且每个项目每个环境唯一；
- `port` 冲突时（同机多执行器）改为各自不同端口并同步告知运维；
- 多网卡机器注册错 IP 时用 `xxl.job.executor.ip` 显式指定。

### 3.2 迁移 Nacos 中心化（目标态）

在 Nacos 新建共享 Data ID `xxl-job.properties`（DEFAULT_GROUP，properties 格式），各接入项目 `spring.config.import` 追加 `- nacos:xxl-job.properties`：

```properties
# xxl-job.properties（Nacos）
xxl.job.admin.addresses=http://<调度中心宿主IP>:8081/xxl-job-admin
xxl.job.accessToken=${nacos.secret.xxl-job.accessToken}
xxl.job.executor.logpath=/app/logs/xxl-job
xxl.job.executor.logretentiondays=30
```

秘密项（存于 Nacos 秘密配置，仅一份）：

```properties
nacos.secret.xxl-job.accessToken=<平台共享令牌-已脱敏>
```

迁移完成后：**删除各项目 yml 中的写死值、轮换令牌**。`appname` 与 `port` 因项目而异，保留在各项目自己的 yml 中，不下沉共享配置。

---

## 4. 执行器配置类

```java
package com.sangerbox.biovisart.config;

import com.xxl.job.core.executor.impl.XxlJobSpringExecutor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * XXL-JOB 执行器配置
 */
@Configuration
public class XxlJobConfig {

    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;

    @Value("${xxl.job.accessToken}")
    private String accessToken;

    @Value("${xxl.job.executor.appname}")
    private String appname;

    @Value("${xxl.job.executor.port}")
    private int port;

    @Value("${xxl.job.executor.logpath}")
    private String logPath;

    @Value("${xxl.job.executor.logretentiondays}")
    private int logRetentionDays;

    @Bean
    public XxlJobSpringExecutor xxlJobExecutor() {
        XxlJobSpringExecutor executor = new XxlJobSpringExecutor();
        executor.setAdminAddresses(adminAddresses);
        executor.setAccessToken(accessToken);
        executor.setAppname(appname);
        executor.setPort(port);
        executor.setLogPath(logPath);
        executor.setLogRetentionDays(logRetentionDays);
        return executor;
    }
}
```

启动日志出现 `>>>>>>>>>>> xxl-job register jobhandler success` 与执行器注册成功日志即接入完成；admin「执行器管理」对应 AppName 的「在线机器」应列出本机 `IP:11011`。

---

## 5. 编写任务（BEAN 模式）

```java
@Slf4j
@Component
public class SampleJobHandler {

    /**
     * 方法名随意，以 @XxlJob 值为调度标识（admin 任务配置的 JobHandler 填它）
     */
    @XxlJob("sampleJobHandler")
    public void sample() {
        XxlJobHelper.log("任务开始，参数：{}", XxlJobHelper.getJobParam());
        try {
            // 业务逻辑
            XxlJobHelper.handleSuccess("处理完成");
        } catch (Exception e) {
            log.error("sample 任务失败", e);
            XxlJobHelper.handleFail("处理失败：" + e.getMessage());
        }
    }
}
```

约定：

- 只用 **BEAN 模式**，禁用 GLUE（Groovy/Shell）模式；
- Handler 命名统一 `xxxHandler`，与 admin 任务配置一一对应；
- 任务方法内打印关键日志用 `XxlJobHelper.log`（会进 admin 调度日志详情页）；
- 任务必须**幂等**：失败重试、人工重触发不应产生副作用叠加；
- 长任务设置 admin 侧「任务超时时间」，阻塞处理策略默认「单机串行（SERIAL_EXECUTION）」。

---

## 6. biovisart-server 存量任务迁移清单

| 现有任务 | 迁移方式 | 说明 |
|---|---|---|
| `AiGalleryReconcileTask.reconcile`（AI 图库对账） | 迁入 xxl-job：去掉 `@Scheduled`，方法加 `@XxlJob("aiGalleryReconcileHandler")` | 集群级幂等任务；cron `0 30 4 * * ?` 配在 admin；顺带解决 dev/prod 双实例重复执行 |
| `NettyCleanUserChannelTask.cleanUserChannel` | 保留 `@Scheduled` | 清理本 JVM 的 `USER_CHANNEL_MAP`，每个实例必须各自执行，不能单点调度 |
| `BvaArtworksServiceImpl.scheduledSyncTask` | 保留 `@Scheduled` | 刷本实例内存 `pendingSyncMap`，单点调度会丢其他实例的待同步数据 |

**判断准则**：操作「集群共享资源」（DB / 对象存储 / 外部接口）的任务进 xxl-job；操作「本实例内存状态」的任务保留本地 `@Scheduled`。

迁移后在 admin 新建任务：

| 配置项 | 值 |
|---|---|
| 执行器 | `biovisart-server-dev` / `-prod` |
| 任务描述 | AI 图库对账 |
| Cron | `0 30 4 * * ?` |
| 运行模式 | BEAN |
| JobHandler | `aiGalleryReconcileHandler` |
| 路由策略 | FIRST（或 ROUND） |
| 阻塞处理 | 单机串行 |
| 超时时间 | 600s |

---

## 7. 联调验证步骤

1. 启动业务应用，确认执行器注册（admin 在线机器列表）；
2. admin 任务列表 → 目标任务 → 「执行一次」（可带参数）；
3. 调度日志：查看触发/执行结果是否为「成功」；失败时点「执行日志」看 `XxlJobHelper.log` 输出与异常栈；
4. 等待一个 cron 周期确认自动调度正常；
5. 停掉一台实例再触发，确认路由到存活实例（验证多实例路由）。

---

## 8. 常见问题

| 问题 | 原因与处理 |
|---|---|
| 执行器不在线 | token 不一致 / 执行器端口（biovisart 为 11011）防火墙未放行 / 注册 IP 误选（见部署文档 §6） |
| job handler not found | admin 填的 JobHandler 与 `@XxlJob` 值不一致 |
| 任务重复执行 | 多环境执行器混注册同一 AppName；或任务未幂等且配置了失败重试 |
| 调度时间差 8 小时 | 执行器 JVM 时区 / admin 容器 TZ 未设置 |
| 想动态传参 | admin 任务「任务参数」或执行一次时填写，代码内 `XxlJobHelper.getJobParam()` 获取 |

---

## 9. 接入 Checklist

- [ ] 依赖 `xxl-job-core:3.4.2` 已引入；
- [ ] `XxlJobConfig` 已添加，配置项齐全（addresses / accessToken / appname / port / logpath / logretentiondays=30）；
- [ ] appname 已在 admin 登记且环境隔离；
- [ ] 存量任务按 §6 准则分类完成迁移/保留；
- [ ] admin 任务已建：cron、路由、阻塞策略、超时齐备；
- [ ] 联调验证 5 步走完；
- [ ] token 写死值已排期迁移 Nacos 并轮换。
