Lab: Distributed Trace - Zipkin
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
git checkout -b my-zipkin-start zipkin-start
```

<!------------------------------------------------------------->
# 2. Zipkin Serverの起動

## TODO 2-01
`platform-services/zipkin` ディレクトリに、 `zipkin.jar` があることを確認してください。

もし無い場合は、 `platform-services/zipkin` ディレクトリで下記のコマンドを実行してください。これにより、 `zipkin.jar` をダウンロードできます。

```bash
curl -sSL https://zipkin.io/quickstart.sh | bash -s
```

## TODO 2-02
下記のコマンドで、実行可能となるようにパーミッションを変更してください。

```bash
chmod 755 ./platform-services/zipkin/zipkin.jar
```

下記のコマンドでZipkin Serverを起動してください。

```bash
java -jar ./platform-services/zipkin/zipkin.jar
```

> ポート番号はデフォルトで9411です。

## TODO 2-03
ブラウザで[Zipkin UI](http://localhost:9411)を開いてください。現時点では、ログ等は受け取っていないため、何も表示されません。

<!------------------------------------------------------------->
# 3. アプリケーションからZipkinへのエクスポート

## TODO 3-01
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
    compile "org.springframework.cloud:spring-cloud-starter-sleuth"

    testCompile project(":components:test-support")

    // この依存性を追加
    compile "org.springframework.cloud:spring-cloud-starter-zipkin"
}
```

## TODO 3-02
`registration-server` および `timesheets-server` 両方の `application.properties` に、下記の設定を追加してください。

```
spring.sleuth.sampler.percentage=1.0
```

> この設定は、全リクエストのどれほど(1.0 = 100%)をZipkin Serverにエクスポートするかを示します。
> デフォルトは0.1(= 10%)です。

## TODO 3-03
`registration-server` および `timesheets-server` を起動してください。

```bash
./gradlew applications:registration-server:bootRun
```

```bash
./gradlew applications:timesheets-server:bootRun
```

下記のcurlコマンドを実行してください。

```bash
curl -i -XPOST -H"Content-Type: application/json" localhost:8084/time-entries/ \
-d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}"
```

ブラウザで[Zipkin UI](http://localhost:9411)を開いて、[Find Traces]ボタンをクリックしてください。そうすると、先ほどのリクエストのトレースが表示されます。

併せて、 `registration-server` および `timesheets-server` のログを確認してください。 `exportable` の値が `true` になっていることが分かります。各ログの最後の部分を見ると、Zipkin Serverにログを転送していることが分かります。

- `registration-server` のログ

```bash
2019-01-16 06:41:42.029 DEBUG [registration-server,,,] 23101 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Received a request to uri [/projects/1] that should not be sampled [false]
2019-01-16 06:41:42.031 DEBUG [registration-server,,,] 23101 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Found a parent span [Trace: bb651192527bb123, Span: a3398af21a07178a, Parent: bb651192527bb123, exportable:true] in the request
2019-01-16 06:41:42.033 DEBUG [registration-server,bb651192527bb123,a3398af21a07178a,true] 23101 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Parent span is [Trace: bb651192527bb123, Span: a3398af21a07178a, Parent: bb651192527bb123, exportable:true]
2019-01-16 06:41:42.043 DEBUG [registration-server,bb651192527bb123,a3398af21a07178a,true] 23101 --- [nio-8083-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Handling span [Trace: bb651192527bb123, Span: a3398af21a07178a, Parent: bb651192527bb123, exportable:true]
2019-01-16 06:41:42.044 DEBUG [registration-server,bb651192527bb123,a3398af21a07178a,true] 23101 --- [nio-8083-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a method tag with value [get] to a span [Trace: bb651192527bb123, Span: a3398af21a07178a, Parent: bb651192527bb123, exportable:true]
2019-01-16 06:41:42.044 DEBUG [registration-server,bb651192527bb123,a3398af21a07178a,true] 23101 --- [nio-8083-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a class tag with value [ProjectController] to a span [Trace: bb651192527bb123, Span: a3398af21a07178a, Parent: bb651192527bb123, exportable:true]
2019-01-16 06:41:43.041  INFO [registration-server,bb651192527bb123,808a8dc60549363e,true] 23101 --- [nio-8083-exec-1] i.p.p.t.projects.ProjectController       : Finding project with ID 1
2019-01-16 06:41:43.043 DEBUG [registration-server,bb651192527bb123,808a8dc60549363e,true] 23101 --- [nio-8083-exec-1] o.s.c.s.zipkin2.DefaultEndpointLocator   : Span will contain serviceName [registration-server]
2019-01-16 06:41:43.077 DEBUG [registration-server,bb651192527bb123,a3398af21a07178a,true] 23101 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Trying to send the parent span [Trace: bb651192527bb123, Span: a3398af21a07178a, Parent: bb651192527bb123, exportable:true] to Zipkin
2019-01-16 06:41:43.078 DEBUG [registration-server,bb651192527bb123,a3398af21a07178a,true] 23101 --- [nio-8083-exec-1] o.s.c.s.zipkin2.DefaultEndpointLocator   : Span will contain serviceName [registration-server]
2019-01-16 06:41:43.080 DEBUG [registration-server,bb651192527bb123,a3398af21a07178a,true] 23101 --- [nio-8083-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Closing the span [Trace: bb651192527bb123, Span: a3398af21a07178a, Parent: bb651192527bb123, exportable:true] since the response was successful
2019-01-16 06:41:43.569 DEBUG [registration-server,,,] 23101 --- [ender@716e431d}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : Created POST request for "http://localhost:9411/api/v2/spans"
2019-01-16 06:41:43.570 DEBUG [registration-server,,,] 23101 --- [ender@716e431d}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : Setting request Accept header to [text/plain, application/json, application/*+json, */*]
2019-01-16 06:41:43.570 DEBUG [registration-server,,,] 23101 --- [ender@716e431d}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : Writing [[B@186df906] as "application/json" using [org.springframework.http.converter.ByteArrayHttpMessageConverter@38641a67]
2019-01-16 06:41:43.580 DEBUG [registration-server,,,] 23101 --- [ender@716e431d}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : POST request for "http://localhost:9411/api/v2/spans" resulted in 202 (Accepted)
```

- `timesheets-server` のログ

```bash
2019-01-16 06:41:40.931 DEBUG [timesheets-server,,,] 23152 --- [nio-8084-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Received a request to uri [/time-entries/] that should not be sampled [false]
2019-01-16 06:41:40.936 DEBUG [timesheets-server,bb651192527bb123,bb651192527bb123,true] 23152 --- [nio-8084-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : No parent span present - creating a new span
2019-01-16 06:41:40.947 DEBUG [timesheets-server,bb651192527bb123,bb651192527bb123,true] 23152 --- [nio-8084-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Handling span [Trace: bb651192527bb123, Span: bb651192527bb123, Parent: null, exportable:true]
2019-01-16 06:41:40.947 DEBUG [timesheets-server,bb651192527bb123,bb651192527bb123,true] 23152 --- [nio-8084-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a method tag with value [create] to a span [Trace: bb651192527bb123, Span: bb651192527bb123, Parent: null, exportable:true]
2019-01-16 06:41:40.947 DEBUG [timesheets-server,bb651192527bb123,bb651192527bb123,true] 23152 --- [nio-8084-exec-1] o.s.c.s.i.web.TraceHandlerInterceptor    : Adding a class tag with value [TimeEntryController] to a span [Trace: bb651192527bb123, Span: bb651192527bb123, Parent: null, exportable:true]
2019-01-16 06:41:41.954 DEBUG [timesheets-server,bb651192527bb123,a3398af21a07178a,true] 23152 --- [nio-8084-exec-1] .w.c.AbstractTraceHttpRequestInterceptor : Starting new client span [[Trace: bb651192527bb123, Span: a3398af21a07178a, Parent: bb651192527bb123, exportable:true]]
2019-01-16 06:41:43.080 DEBUG [timesheets-server,bb651192527bb123,a3398af21a07178a,true] 23152 --- [nio-8084-exec-1] o.s.c.s.zipkin2.DefaultEndpointLocator   : Span will contain serviceName [timesheets-server]
2019-01-16 06:41:43.121 DEBUG [timesheets-server,bb651192527bb123,bb651192527bb123,true] 23152 --- [nio-8084-exec-1] o.s.c.sleuth.instrument.web.TraceFilter  : Closing the span [Trace: bb651192527bb123, Span: bb651192527bb123, Parent: null, exportable:true] since the response was successful
2019-01-16 06:41:43.121 DEBUG [timesheets-server,bb651192527bb123,bb651192527bb123,true] 23152 --- [nio-8084-exec-1] o.s.c.s.zipkin2.DefaultEndpointLocator   : Span will contain serviceName [timesheets-server]
2019-01-16 06:41:43.123 DEBUG [timesheets-server,,,] 23152 --- [ender@163042ea}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : Created POST request for "http://localhost:9411/api/v2/spans"
2019-01-16 06:41:43.123 DEBUG [timesheets-server,,,] 23152 --- [ender@163042ea}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : Setting request Accept header to [text/plain, application/json, application/*+json, */*]
2019-01-16 06:41:43.124 DEBUG [timesheets-server,,,] 23152 --- [ender@163042ea}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : Writing [[B@29798571] as "application/json" using [org.springframework.http.converter.ByteArrayHttpMessageConverter@4a91a878]
2019-01-16 06:41:43.159 DEBUG [timesheets-server,,,] 23152 --- [ender@163042ea}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : POST request for "http://localhost:9411/api/v2/spans" resulted in 202 (Accepted)
2019-01-16 06:41:44.164 DEBUG [timesheets-server,,,] 23152 --- [ender@163042ea}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : Created POST request for "http://localhost:9411/api/v2/spans"
2019-01-16 06:41:44.165 DEBUG [timesheets-server,,,] 23152 --- [ender@163042ea}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : Setting request Accept header to [text/plain, application/json, application/*+json, */*]
2019-01-16 06:41:44.165 DEBUG [timesheets-server,,,] 23152 --- [ender@163042ea}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : Writing [[B@42a665e8] as "application/json" using [org.springframework.http.converter.ByteArrayHttpMessageConverter@4a91a878]
2019-01-16 06:41:44.167 DEBUG [timesheets-server,,,] 23152 --- [ender@163042ea}] o.s.c.s.z.s.ZipkinRestTemplateWrapper    : POST request for "http://localhost:9411/api/v2/spans" resulted in 202 (Accepted)
```

<!------------------------------------------------------------->
# 4. トレースのストリーミング

アプリケーションに `spring-cloud-starter-zipkin` を追加した場合、ログはHTTPでZipkin Serverに送信されます。HTTPでログを同期的に送信すると、パフォーマンスに影響が出る可能性があります（特に、大量にトレーシングしている場合）。その場合、RabbitMQなどのメッセージキューを利用して、非同期でログを送信します。

## TODO 4-01
下記のコマンドでRabbitMQを起動してください。

### 直接インストールした場合

```bash
rabbitmq-server
```

### Dockerを使っている場合

```bash
docker container start rabbitmq-scd
```

## TODO 4-02
ブラウザで[RabbitMQ管理コンソール](http://localhost:15672/)を開いてください。

> ユーザー名 `guest` 、パスワード `guest` です。

### Queueの新規作成
1. [Queuesタブ](http://localhost:15672/#/queues)を開いてください。
2. [Add a new queue]の[Name]欄に `zipkin` と入力してください。
3. [Add queue]ボタンをクリックしてください。

> `zipkin` は、Zipkin Serverがデフォルトで利用するキューの名前です。

## TODO 4-03
`registration-server` および `timesheets-server` を停止してください。

## TODO 4-04
`applications/server.gradle` を下記のように変更してください。

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
    compile "org.springframework.cloud:spring-cloud-starter-sleuth"

    testCompile project(":components:test-support")

    compile "org.springframework.cloud:spring-cloud-starter-zipkin"

    // この依存性を追加
    compile "org.springframework.amqp:spring-rabbit"
}
```

## TODO 4-05
`registration-server` および `timesheets-server` を再起動してください。

```bash
./gradlew applications:registration-server:bootRun
```

```bash
./gradlew applications:timesheets-server:bootRun
```

## TODO 4-06
Zipkin Serverを停止してください。

## TODO 4-07
下記のコマンドで、Zipkin Serverを再起動してください。

```bash
RABBIT_URI=amqp://guest:guest@localhost:5672 java -jar platform-services/zipkin/zipkin.jar
```

> 詳細な設定は[こちら](https://github.com/openzipkin/zipkin/tree/master/zipkin-collector/rabbitmq)を参照してください。

## TODO 4-08
下記のcurlコマンドを実行してください。

```bash
curl -i -XPOST -H"Content-Type: application/json" localhost:8084/time-entries/ \
-d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}"
```

## TODO 4-09
ブラウザで[Zipkin UI](http://localhost:9411)を開いて、[Find Traces]ボタンをクリックしてください。そうすると、先ほどのリクエストのトレースが表示されます。

> ここからの見た目は変わりませんが、ログはRabbitMQを経由して取得したものです。

## TODO 4-10
RabbitMQ管理コンソールで[zipkinキュー](http://localhost:15672/#/queues/%2F/zipkin)を確認してください。[Overview]-[Message rates]を見ると、メッセージを受信していることが分かります。

<!------------------------------------------------------------->
# 5. 後片付け
- 全クラスを停止してください。
- RabbitMQを `docker container stop rabbitmq-scd` で停止してください。
- 下記のコマンドでコミットしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git add .
git commit -m "completed"
```

> 未コミットのまま次の演習でブランチを切り替えると、新しいブランチにも影響が出てしまいます。必ずコミットを行なってください。

この演習は以上です。