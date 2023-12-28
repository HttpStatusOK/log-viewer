# log-viewer
基于 Springboot logback + slf4j 的简易日志查看器

<img width="1716" alt="image" src="https://github.com/HttpStatusOK/log-viewer/assets/116876598/1f044563-8c66-47dd-ab06-7653e010fcd7">

错误栈信息查看

<img width="1721" alt="image" src="https://github.com/HttpStatusOK/log-viewer/assets/116876598/e6f68b23-242d-4f19-9a38-1b2574896087">

# 应用场景
- 小公司小项目，没有日志收集
- 懒得上服务器 tail log

# 运行逻辑

1. SpringBoot Logback 日志配置文件滚动输出
2. 做两条 API：一条用来读取日志文件夹下的文件列表，一条用来读取日志文件，然后解析返回给前端
3. 前度根据两条 API 读取相关内容

# 配置 Logback

classpath 下创建 bogback-spring.xml 配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true">

    <!-- console 日志输出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS,GMT+8} [%X{requestId}%X{uid}] %level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 日志文件滚动输出 -->
    <appender name="LOG_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS,GMT+8} [%X{requestId}%X{uid}] %level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <!-- 按日期分割，最大保留 60 天，一个文件最大 100MB -->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>/root/log/app-%d{yyyy-MM-dd}-%i.log</fileNamePattern>
            <MaxHistory>60</MaxHistory>
            <maxFileSize>100MB</maxFileSize>
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <!-- DEV 环境下用 console 输出 -->
    <springProfile name="dev">
        <root level="INFO">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>

    <!-- PROD 环境下用文件输出 -->
    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="LOG_FILE" />
        </root>
    </springProfile>

</configuration>
```

配置 requestId、uid，这里按业务需求配置就好了，方便定位日志

```java
@Component
public class LoggerConfiguration implements Filter {

   @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws ServletException, IOException {
        try {
            String id = null;
            try {
                id = StpUtil.getLoginIdAsString();
            } catch (Exception ignore) {}
            if (StringUtils.isNotEmpty(id)) {
                MDC.put("uid", " #" + id);
            }
            HttpServletRequest req = (HttpServletRequest) request;
            HttpServletResponse resp = (HttpServletResponse) response;
            String requestId = req.getHeader("X-Request-Id");
            if (StringUtils.isEmpty(requestId)) {
                requestId = UUID.randomUUID().toString().replaceAll("-", "").toLowerCase();
            }
            MDC.put("requestId", requestId);
            resp.setHeader("X-Request-Id", requestId);
        } catch (Exception ignore) {}
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }

}

```
