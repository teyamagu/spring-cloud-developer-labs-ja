Lab: Distributed Trace - Sleuth
======================================

# この演習の目標
- Spring Cloud Sluethを利用して、ログに `traceId` や `spanId` を出力します。

# 目標時間
- 30分間

# 1. 準備

## TODO 1-01
下記コマンドで今回の演習用ブランチを作成・切り替えしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git checkout -b my-sleuth-start sleuth-start
```

<!------------------------------------------------------------->
# 2. Producerの作成

## TODO 2-01
`applications/server.gradle` を下記のように編集してください。

```groovy
apply plugin: "org.springframework.boot"

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:$springCloudVersion"
    }
}

dependencies {
    compile project(":components:rest-support")

    compile "org.springframework.boot:spring-boot-starter-web"
    compile "org.springframework.boot:spring-boot-starter-actuator"

    testCompile project(":components:test-support")

    // この依存性を追加
    compile "org.springframework.cloud:spring-cloud-starter-sleuth"
}
```

## TODO 2-02
`applications` 内の `registration-server` および `timesheets-server` 両方の `application.properties` に、下記の記述を追加してください。

```
logging.level.org.springframework.cloud.sleuth=DEBUG
```

## TODO 2-03
下記のコマンドで、 `registration-server` および `timesheets-server` を起動してください。

```bash
./gradlew applications:registration-server:bootRun
```

```bash
./gradlew applications:timesheets-server:bootRun
```

## TODO 2-04
下記のcurlコマンドで `timesheets-server` にアクセスしてください。

```bash
curl -i -XPOST -H"Content-Type: application/json" localhost:8084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}"
```

## TODO 2-05
`timesheets-server` のログを確認してください。下記のようになっています。

```bash
2018-03-13 16:11:55.145 DEBUG [timesheets-server,5c2ff6328c05973b,5c2ff6328c05973b,false] 85437 --- [nio-8084-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : No parent span present - creating a new span
2018-03-13 16:11:55.157 DEBUG [timesheets-server,5c2ff6328c05973b,5c2ff6328c05973b,false] 85437 --- [nio-8084-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Handling span [Trace: 5c2ff6328c05973b, Span: 5c2ff6328c05973b, Parent: null, exportable:false]
2018-03-13 16:11:55.158 DEBUG [timesheets-server,5c2ff6328c05973b,5c2ff6328c05973b,false] 85437 --- [nio-8084-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a method tag with value [create] to a span [Trace: 5c2ff6328c05973b, Span: 5c2ff6328c05973b, Parent: null, exportable:false]
2018-03-13 16:11:55.158 DEBUG [timesheets-server,5c2ff6328c05973b,5c2ff6328c05973b,false] 85437 --- [nio-8084-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a class tag with value [TimeEntryController] to a span [Trace: 5c2ff6328c05973b, Span: 5c2ff6328c05973b, Parent: null, exportable:false]
2018-03-13 16:11:55.671 DEBUG [timesheets-server,5c2ff6328c05973b,39f834359f48ced5,false] 85437 --- [nio-8084-exec-1] .w.c.AbstractTraceHttpRequestInterceptor : Starting new client span [[Trace: 5c2ff6328c05973b, Span: 39f834359f48ced5, Parent: 5c2ff6328c05973b, exportable:false]]
2018-03-13 16:11:56.101 DEBUG [timesheets-server,5c2ff6328c05973b,5c2ff6328c05973b,false] 85437 --- [nio-8084-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Closing the span [Trace: 5c2ff6328c05973b, Span: 5c2ff6328c05973b, Parent: null, exportable:false] since the response was successful
```

## TODO 2-06
`registration-server` のログを確認してください。下記のようになっています。

```bash
2018-03-13 16:11:55.780 DEBUG [registration-server,,,] 85455 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Found a parent span [Trace: 5c2ff6328c05973b, Span: 39f834359f48ced5, Parent: 5c2ff6328c05973b, exportable:false] in the request
2018-03-13 16:11:55.782 DEBUG [registration-server,5c2ff6328c05973b,39f834359f48ced5,false] 85455 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Parent span is [Trace: 5c2ff6328c05973b, Span: 39f834359f48ced5, Parent: 5c2ff6328c05973b, exportable:false]
2018-03-13 16:11:55.797 DEBUG [registration-server,5c2ff6328c05973b,39f834359f48ced5,false] 85455 --- [nio-8083-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Handling span [Trace: 5c2ff6328c05973b, Span: 39f834359f48ced5, Parent: 5c2ff6328c05973b, exportable:false]
2018-03-13 16:11:55.798 DEBUG [registration-server,5c2ff6328c05973b,39f834359f48ced5,false] 85455 --- [nio-8083-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a method tag with value [get] to a span [Trace: 5c2ff6328c05973b, Span: 39f834359f48ced5, Parent: 5c2ff6328c05973b, exportable:false]
2018-03-13 16:11:55.798 DEBUG [registration-server,5c2ff6328c05973b,39f834359f48ced5,false] 85455 --- [nio-8083-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a class tag with value [ProjectController] to a span [Trace: 5c2ff6328c05973b, Span: 39f834359f48ced5, Parent: 5c2ff6328c05973b, exportable:false]
2018-03-13 16:11:56.067 DEBUG [registration-server,5c2ff6328c05973b,39f834359f48ced5,false] 85455 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Trying to send the parent span [Trace: 5c2ff6328c05973b, Span: 39f834359f48ced5, Parent: 5c2ff6328c05973b, exportable:false] to Zipkin
2018-03-13 16:11:56.068 DEBUG [registration-server,5c2ff6328c05973b,39f834359f48ced5,false] 85455 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Closing the span [Trace: 5c2ff6328c05973b, Span: 39f834359f48ced5, Parent: 5c2ff6328c05973b, exportable:false] since the response was successful
```

Spring Cloud Sleuthによるログフォーマットは下記のようになっています。

```bash
[appname,traceId,spanId,exportable]
```

1. `appname` : アプリケーション名
2. `traceId` : このSpanが含まれるTraceのID
3. `spanId` : このSpanのID
4. `exportable` : ログがZipkin（次章）にエクスポートできるかどうか

`timesheets-server` と `registration-server` のログを読み、上記の値がどのようになっているか確認してください。

> 次章では、Zipkinと連携するライブラリも依存性に追加します。その際は `exportable` が `true` になります。

## TODO 2-A1 (オプション)
`registration-server` に下記のような `io.pivotal.pal.tracker.registration.LoggingFilter` クラスを作成してください。

```java
package io.pivotal.pal.tracker.registration;

import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.Enumeration;

public class LoggingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, FilterChain filterChain)
            throws ServletException, IOException {
        System.out.println("======================================");
        System.out.println("## REQUEST");
        String requestMethodAndUri = httpServletRequest.getMethod() + " " + httpServletRequest.getRequestURI();
        System.out.println(requestMethodAndUri);
        for (Enumeration<String> headerNames = httpServletRequest.getHeaderNames(); headerNames.hasMoreElements();) {
            String headerName = headerNames.nextElement();
            String headerValue = httpServletRequest.getHeader(headerName);
            System.out.println(headerName + ": " + headerValue);
        }
        System.out.println("======================================");
        filterChain.doFilter(httpServletRequest, httpServletResponse);
        System.out.println("======================================");
        System.out.println("## RESPONSE (for " + requestMethodAndUri + ")");
        System.out.println(httpServletResponse.getStatus());
        for (String headerName : httpServletResponse.getHeaderNames()) {
            String headerValue = httpServletResponse.getHeader(headerName);
            System.out.println(headerName + ": " + headerValue);
        }
        System.out.println("======================================");
    }
}
```

## TODO 2-A2 (オプション)
`registration-server` の `RegistrationServerApp` クラスを下記のように編集してください。

```java
package io.pivotal.pal.tracker.registration;

import io.pivotal.pal.tracker.projects.data.ProjectDataGateway;
import io.pivotal.pal.tracker.projects.data.ProjectFields;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;

import javax.annotation.PostConstruct;
import java.util.TimeZone;

import static io.pivotal.pal.tracker.projects.data.ProjectFields.projectFieldsBuilder;


@SpringBootApplication
@ComponentScan({
    "io.pivotal.pal.tracker.accounts",
    "io.pivotal.pal.tracker.restsupport",
    "io.pivotal.pal.tracker.projects",
    "io.pivotal.pal.tracker.users"
})
public class RegistrationApp {
    public static void main(String[] args) {
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
        SpringApplication.run(RegistrationApp.class, args);
    }

    private final ProjectDataGateway gateway;
    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    public RegistrationApp(ProjectDataGateway gateway) {
        this.gateway = gateway;
    }

    @PostConstruct
    public void init(){
        // Make sure there is data in the registration server when
        // it starts.
        ProjectFields project = projectFieldsBuilder()
                                    .accountId(1)
                                    .name("Basket Weaving")
                                    .build();
        logger.info("**********************************");
        logger.info("Creating project: " + project);
        logger.info("**********************************");
        gateway.create(project);
    }

    /*
     * このメソッドを追加
     */
    @Bean
    public FilterRegistrationBean loggingFilter() {
        LoggingFilter loggingFilter = new LoggingFilter();
        FilterRegistrationBean registrationBean = new FilterRegistrationBean(loggingFilter);
        registrationBean.setOrder(Integer.MIN_VALUE);
        registrationBean.addUrlPatterns("/*");
        return registrationBean;
    }
}
```

## TODO 2-A3 (オプション)
`registration-server` を再起動後、下記のcurlコマンドで `timesheets-server` にアクセスしてください。

```bash
curl -i -XPOST -H"Content-Type: application/json" localhost:8084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}"
```

## TODO 2-A4 (オプション)
`registration-server` のログを確認してください。リクエストおよびレスポンスのヘッダーがログに出力されています。

- `timesheets-server` のログ

```bash
2019-01-16 05:04:32.216 DEBUG [timesheets-server,,,] 17478 --- [nio-8084-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Received a request to uri [/time-entries/] that should not be sampled [false]
2019-01-16 05:04:32.222 DEBUG [timesheets-server,7cff49edf375b638,7cff49edf375b638,false] 17478 --- [nio-8084-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : No parent span present - creating a new span
2019-01-16 05:04:32.231 DEBUG [timesheets-server,7cff49edf375b638,7cff49edf375b638,false] 17478 --- [nio-8084-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Handling span [Trace: 7cff49edf375b638, Span: 7cff49edf375b638, Parent: null, exportable:false]
2019-01-16 05:04:32.231 DEBUG [timesheets-server,7cff49edf375b638,7cff49edf375b638,false] 17478 --- [nio-8084-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a method tag with value [create] to a span [Trace: 7cff49edf375b638, Span: 7cff49edf375b638, Parent: null, exportable:false]
2019-01-16 05:04:32.231 DEBUG [timesheets-server,7cff49edf375b638,7cff49edf375b638,false] 17478 --- [nio-8084-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a class tag with value [TimeEntryController] to a span [Trace: 7cff49edf375b638, Span: 7cff49edf375b638, Parent: null, exportable:false]
2019-01-16 05:04:32.827 DEBUG [timesheets-server,7cff49edf375b638,8b435d6761214f0b,false] 17478 --- [nio-8084-exec-1] .w.c.AbstractTraceHttpRequestInterceptor : Starting new client span [[Trace: 7cff49edf375b638, Span: 8b435d6761214f0b, Parent: 7cff49edf375b638, exportable:false]]
2019-01-16 05:04:33.957 DEBUG [timesheets-server,7cff49edf375b638,7cff49edf375b638,false] 17478 --- [nio-8084-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Closing the span [Trace: 7cff49edf375b638, Span: 7cff49edf375b638, Parent: null, exportable:false] since the response was successful
```

- `registration-server` のログ

```bash
======================================
## REQUEST
GET /projects/1
accept: application/json, application/*+json
x-b3-traceid: 7cff49edf375b638
x-b3-spanid: 8b435d6761214f0b
x-b3-sampled: 0
x-span-name: http:/projects/1
x-b3-parentspanid: 7cff49edf375b638
user-agent: Java/1.8.0_191
host: localhost:8083
connection: keep-alive
======================================
2019-01-16 05:04:32.914 DEBUG [registration-server,,,] 17461 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Received a request to uri [/projects/1] that should not be sampled [true]
2019-01-16 05:04:32.917 DEBUG [registration-server,,,] 17461 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Found a parent span [Trace: 7cff49edf375b638, Span: 8b435d6761214f0b, Parent: 7cff49edf375b638, exportable:false] in the request
2019-01-16 05:04:32.919 DEBUG [registration-server,7cff49edf375b638,8b435d6761214f0b,false] 17461 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Parent span is [Trace: 7cff49edf375b638, Span: 8b435d6761214f0b, Parent: 7cff49edf375b638, exportable:false]
2019-01-16 05:04:32.930 DEBUG [registration-server,7cff49edf375b638,8b435d6761214f0b,false] 17461 --- [nio-8083-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Handling span [Trace: 7cff49edf375b638, Span: 8b435d6761214f0b, Parent: 7cff49edf375b638, exportable:false]
2019-01-16 05:04:32.931 DEBUG [registration-server,7cff49edf375b638,8b435d6761214f0b,false] 17461 --- [nio-8083-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a method tag with value [get] to a span [Trace: 7cff49edf375b638, Span: 8b435d6761214f0b, Parent: 7cff49edf375b638, exportable:false]
2019-01-16 05:04:32.931 DEBUG [registration-server,7cff49edf375b638,8b435d6761214f0b,false] 17461 --- [nio-8083-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a class tag with value [ProjectController] to a span [Trace: 7cff49edf375b638, Span: 8b435d6761214f0b, Parent: 7cff49edf375b638, exportable:false]
2019-01-16 05:04:33.933 DEBUG [registration-server,7cff49edf375b638,8b435d6761214f0b,false] 17461 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Trying to send the parent span [Trace: 7cff49edf375b638, Span: 8b435d6761214f0b, Parent: 7cff49edf375b638, exportable:false] to Zipkin
2019-01-16 05:04:33.934 DEBUG [registration-server,7cff49edf375b638,8b435d6761214f0b,false] 17461 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Closing the span [Trace: 7cff49edf375b638, Span: 8b435d6761214f0b, Parent: 7cff49edf375b638, exportable:false] since the response was successful
======================================
## RESPONSE (for GET /projects/1)
200
X-Application-Context: registration-server:8083
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Wed, 16 Jan 2019 05:04:33 GMT
======================================
```

> `traceId` や呼び出し元マイクロサービスの `spanId` は、リクエストヘッダーで伝搬します。

<!------------------------------------------------------------->
# 3. Spanの自作

## TODO 3-01
`components/projects` の `build.gradle` を、下記のように編集してください。

```groovy
dependencies {
    compile project(":components:rest-support")

    testCompile project(":components:test-support")

    // これらの依存性を追加
    compile "org.springframework.cloud:spring-cloud-sleuth-core:1.3.2.RELEASE"
    compile "org.slf4j:slf4j-api:1.7.25"
}
```

## TODO 3-02
`components/projects` の `ProjectController` クラスを下記のように編集してください。

```java
package io.pivotal.pal.tracker.projects;

import io.pivotal.pal.tracker.projects.data.ProjectDataGateway;
import io.pivotal.pal.tracker.projects.data.ProjectFields;
import io.pivotal.pal.tracker.projects.data.ProjectRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.sleuth.Tracer;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

import static io.pivotal.pal.tracker.projects.ProjectInfo.projectInfoBuilder;
import static io.pivotal.pal.tracker.projects.data.ProjectFields.projectFieldsBuilder;
import static io.pivotal.pal.tracker.restsupport.InjectDelay.injectDelay;
import static java.util.stream.Collectors.toList;

@RestController
@RequestMapping("/projects")
public class ProjectController {

    // ロガーを追加
    private final Logger log = LoggerFactory.getLogger(ProjectController.class);

    private final ProjectDataGateway gateway;

    // Tracerを追加
    private final Tracer tracer;

    // コンストラクタにTracerの引数を追加
    public ProjectController(ProjectDataGateway gateway, Tracer tracer) {
        this.gateway = gateway;
        this.tracer = tracer;
    }
    
    // 以下省略
```

> Spring Bootでは、 `org.springframework.cloud.sleuth.Tracer` のBeanが自動登録されます。

## TODO 3-03
`ProjectController` クラスの `get` メソッドを下記のように編集してください。

```java
    @GetMapping("/{projectId}")
    public ProjectInfo get(@PathVariable long projectId) {
        injectDelay(1L);

        // 追加
        Span findProjectSpan = this.tracer.createSpan("ProjectController.findProject");

        // 追加
        log.info("Finding project with ID {}", projectId);

        ProjectRecord record = gateway.find(projectId);

        // 追加
        this.tracer.close(findProjectSpan);

        if (record != null) {
            return present(record);
        }

        return null;
    }
```

`registration-server` を再起動してください。

```bash
./gradlew applications:registration-server:bootRun
```

curlコマンドでアクセスしてください。

```bash
curl -i -XPOST -H"Content-Type: application/json" localhost:8084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}"
```

`registration-server` のログを確認してください。

`Finding project with ID 1` のログだけ `spanId` が変わっており、別のSpanになっていることが分かります。

```bash
...
2019-01-16 05:24:25.836 DEBUG [registration-server,774d26a6fab4245d,3604ed2626e5dbf0,false] 18987 --- [nio-8083-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a class tag with value [ProjectController] to a span [Trace: 774d26a6fab4245d, Span: 3604ed2626e5dbf0, Parent: 774d26a6fab4245d, exportable:false]
2019-01-16 05:24:26.196  INFO [registration-server,774d26a6fab4245d,47078ef3ccd6fb8a,false] 18987 --- [nio-8083-exec-1] i.p.p.t.projects.ProjectController       : Finding project with ID 1
2019-01-16 05:24:26.228 DEBUG [registration-server,774d26a6fab4245d,3604ed2626e5dbf0,false] 18987 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Trying to send the parent span [Trace: 774d26a6fab4245d, Span: 3604ed2626e5dbf0, Parent: 774d26a6fab4245d, exportable:false] to Zipkin
...
```

<!------------------------------------------------------------->
# 4. 後片付け
- 全クラスを停止してください。
- 下記のコマンドでコミットしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git add .
git commit -m "completed"
```

> 未コミットのまま次の演習でブランチを切り替えると、新しいブランチにも影響が出てしまいます。必ずコミットを行なってください。

この演習は以上です。