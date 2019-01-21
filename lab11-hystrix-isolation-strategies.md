Lab: Hystrix Isolation Strategies
======================================

# この演習の目標
- スレッドプールとセマフォの違いを理解します。

# 目標時間
- 30分間

# 1. 準備

## TODO 1-01
下記コマンドで今回の演習用ブランチを作成・切り替えしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git checkout -b my-hystrix-isolation-strategies-start hystrix-isolation-strategies-start
```

<!------------------------------------------------------------->
# 2. JMeterの準備

## TODO 2-01
`spring-cloud-developer-code/scripts` ディレクトリに、 `CreateTimeEntryTest.jmx` というファイルが存在することを確認してください。

後ほど、JMeterでアプリケーションに負荷をかけるために利用します。

<!------------------------------------------------------------->
# 3. Hystrixの設定

## TODO 3-01
`components/timesheets` フォルダの `ProjectClient` クラスを見てください（変更不要）。

- `getProject` には `threadPoolKey` が指定されていません。この場合、Hystrixデフォルトのスレッドプールが使われます。
- `getProjectFromCache` は `ProjectClientCache` というスレッドプールで実行され、 `getProject` とは隔離されます。

> `commandKey` 要素はこのコマンドの名前を指定します。指定しなかった場合、メソッド名が使われます。

## TODO 3-02
`applications/timesheets-server` の `application.properties` を見てください（変更不要）。

`hystrix.command.default` で始まる3つのHystrix関連プロパティが指定されています。

- `hystrix.command.default.execution.isolation.strategy=SEMAPHORE`
  - 隔離戦略をセマフォに指定しています（現時点ではコメントアウトされています）
- `hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=2000`
  - タイムアウト時間を2秒に指定しています 
- `hystrix.command.default.circuitBreaker.requestVolumeThreshold=2`
  - 10秒間のあいだに2回以上のリクエストでOpen状態になるかどうかを検討しはじめます

<!------------------------------------------------------------->
# 4. レイテンシの注入

## TODO 4-01
`components/instrumentation` を確認してください（変更不要）。AOPを利用して、何秒間かスリープ処理を注入しています。

- `FixedLatencyCmd` クラス
  - スリープ処理を行うクラスです。
- `LatencyCmd` インタフェース
  - `FixedLatencyCmd` クラスが実装しているインタフェースです。
- `InstrumentLatency` アノテーション
  - スリープ処理を注入する対象メソッドに付加するアノテーションです。
- `InjectLatencyAspect` クラス
  - スリープ処理を注入するアスペクトクラスです。
  - `InstrumentLatency` アノテーションが付加されたメソッドに割り込み処理を行うよう記述されています。

## TODO 4-02
`applications/registration-server` の `InstrumentedLatencyConfig` クラスを確認してください（変更不要）。

`FixedLatencyCmd` と `InjectLatencyAspect` がBean定義されています。

`registration.server.latency.ms` の値は `application.properties` に `100` （単位はミリ秒）と指定されています。

## TODO 4-03
`components/projects` の `ProjectController` クラスを確認してください（変更不要）。

`get` メソッドに `InstrumentLatency` アノテーションが付加されていることが分かります。これにより、このメソッドには100秒のスリープ処理が割り込まれます。

## TODO 4-04
hystrix-dashboard、registration-server、timesheets-serverを起動してください。

<!------------------------------------------------------------->
# 5. スレッドプールでの実行準備

## TODO 5-01
Hystrix Dashboardでtimesheets-serverを監視してください（URLは[こちら](http://localhost:8085/hystrix/monitor?stream=http%3A%2F%2Flocalhost%3A8084%2Fhystrix.stream&title=Timesheets%20Application)）。

## TODO 5-02
ターミナルで `scripts` ディレクトリに移動し、下記のコマンドでJMeterを実行してください。これにより、JMeterからtimesheest-serverに2リクエスト/秒を送信し、負荷をかけています。

```bash
jmeter -n -t CreateTimeEntryTest.jmx
```

## TODO 5-03
Hystrix Dashboardを確認してください。Openになったりエラーが出たりはしていません。

<!------------------------------------------------------------->
# 6. スレッドのスタックをシミュレーションする

## TODO 6-01
`applications/registration-server` の `application.properties` を下記のように変更してください。スリープ時間を1000秒にしてタイムアウトが発生するようにします。

```
spring.application.name=registration-server

server.port=8083

# Change this property
registration.server.latency.ms=1000000

management.security.enabled=false
```

## TODO 6-02
`applications/registration-server` を再起動してください。

## TODO 6-03
Hystrix Dashboardを確認してください。

- `ProjectClient` のCircuitがOpenになります。全リクエストでタイムアウトが発生したためです。
  - タイムアウト時間は、 `application.properties` で2秒に設定されています。
- しばらくすると、ほとんどの経過時間が0msになります。これは、Open状態ではすぐにフォールバックが実行され、registration-serverへのリクエストが行われないためです。
- たまに経過時間が長くなっているのは、5秒毎にHalf Open状態になり、registration-serverへのリクエストが行われるためです。しかし、またタイムアウトが発生するため、すぐにOpen状態に戻ります。これが繰り返されます。

<!------------------------------------------------------------->
# 7. 信頼できないクライアントでのセマフォの利用

## TODO 7-01
`applications/registration-server` の `application.properties` を下記のように変更してください。スリープ時間を0.1秒に戻します。

```
spring.application.name=registration-server

server.port=8083

# Change this property
registration.server.latency.ms=100

management.security.enabled=false
```

## TODO 7-02
`applications/registration-server` を再起動してください。

## TODO 7-03
`applications/timesheets-server` の `application.properties` を下記のように変更してください。Hystrixの隔離戦略をセマフォに変更します。

```
spring.application.name=timesheets-server

server.port=8084
registration.server.endpoint=http://localhost:8083
management.security.enabled=false

# Uncomment this line
hystrix.command.default.execution.isolation.strategy=SEMAPHORE

hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=2000

hystrix.command.default.circuitBreaker.requestVolumeThreshold=2
```

## TODO 7-04
`applications/timesheets-server` を再起動してください。

## TODO 7-05
JMeterを実行してください。

> 以前のTODOから起動したままであれば、そのままでOKです（再起動の必要はありません）。

## TODO 7-06
Hystrix Dashboardを確認してください。スレッドプールを使っていないため、画面下部には何も表示されません。

## TODO 7-07
`applications/registration-server` の `application.properties` を下記のように変更してください。スリープ時間を1000秒にしてタイムアウトが発生するようにします。

```
spring.application.name=registration-server

server.port=8083

# Change this property
registration.server.latency.ms=1000000

management.security.enabled=false
```

## TODO 7-08
`applications/registration-server` を再起動してください。

## TODO 7-09
Hystrix Dashboardを確認してください。

- `ProjectClient` のCircuitはOpen状態になっています。レスポンス時間は約2000msです。
- `ProjectClientCache` のCircuitはClose状態になっています。レスポンス時間は約1msです。

## TODO 7-10
JMeterのコンソール出力を確認してください。エラーが発生している（ `Err` が0以外の値になっている）はずです。

## TODO 7-11
http://localhost:8084/metrics にブラウザでアクセスし、 `threads` の数を確認してください。1分待った後、ブラウザをリロードしてください。 `threads` の数が変わったはずです。

これは、セマフォがスレッドのスタックを防いでいないことを示しています。

- サーキットブレイカーがHalf Open状態での最初に実行されるリクエストは、Tomcatスレッドプールを枯渇させます。
- JMeterからtimesheets-serverへの最初のリクエストは、JMeterクライアントソケット接続でタイムアウトとなり、エラーが表示されます。
- テストの実行を続けていると、Tomcatはスレッドを使い果たし、再起動するまでtimesheets-serverはハングし続けます。

## TODO 7-12
JMeterを停止してください。

<!------------------------------------------------------------->
# 8. 信頼できるクライアントでのセマフォの利用

## TODO 8-01
`ProjectClient` にソケットタイムアウトを設定するために、`applications/timesheets-server` の `application.properties` を下記のように編集してください。

```
spring.application.name=timesheets-server

server.port=8084
registration.server.endpoint=http://localhost:8083
management.security.enabled=false

## Project Client Socket Timeout Config
project.client.connect.timeout.ms=500
project.client.read.timeout.ms=1000

## Hystrix Configuration
hystrix.command.default.execution.isolation.strategy=SEMAPHORE
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=2000
hystrix.command.default.circuitBreaker.requestVolumeThreshold=2
```

## TODO 8-02
`timesheets-server` の `TimesheetsApp` クラスを下記のように編集してください。 `RestOperations` Beanを上書きして、ソケットタイムアウトのパラメーターを受け取るようにしてください。

```java
package io.pivotal.pal.tracker.timesheets;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Primary;
import org.springframework.web.client.RestOperations;

import java.util.TimeZone;


@SpringBootApplication
@EnableCircuitBreaker
@ComponentScan({"io.pivotal.pal.tracker.timesheets", "io.pivotal.pal.tracker.restsupport"})
public class TimesheetsApp {

    public static void main(String[] args) {
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
        SpringApplication.run(TimesheetsApp.class, args);
    }

    // NOTE: We are ignoring the Rest Support Rest Operations
    // given we must apply RestTemplateBuilder via Spring Boot
    // Dependencies
    @Primary
    @Bean
    RestOperations restOperations(
            @Value("${project.client.connect.timeout.ms}") int connectTimeout,
            @Value("${project.client.read.timeout.ms}") int readTimeout
    ) {
        return new RestTemplateBuilder()
                .setConnectTimeout(connectTimeout)
                .setReadTimeout(readTimeout)
                .build();
    }

    @Bean
    ProjectClient projectClient(
        RestOperations restOperations,
        @Value("${registration.server.endpoint}") String registrationEndpoint
    ) {
        return new ProjectClient(restOperations, registrationEndpoint);
    }
}
```

> `RestTemplateBuilder` クラスは、Spring Bootで定義されています。その名の通り、様々なパラメーターを受け取って `RestTemplate` を生成するクラスです。

## TODO 8-03
`timesheets-server` を再起動してください。

## TODO 8-04
`applications/registration-server` の `application.properties` を下記のように変更し、スリープ時間を100ミリ秒に設定してください。

```
spring.application.name=registration-server

server.port=8083

# Change this property
registration.server.latency.ms=100

management.security.enabled=false
```

変更を保存したら、 `registration-server` を再起動してください。

<!------------------------------------------------------------->
# 9. テストの再実行
## TODO 9-01
JMeterを起動してください。

## TODO 9-02
[Hystrix Dashboard](http://localhost:8085/hystrix/monitor?stream=http%3A%2F%2Flocalhost%3A8084%2Fhystrix.stream)を確認してください。エラーは表示されていないことが分かります。

## TODO 9-03
`applications/registration-server` の `application.properties` を下記のように変更し、スリープ時間を1000000ミリ秒に設定してください。

```
spring.application.name=registration-server

server.port=8083

# Change this property
registration.server.latency.ms=1000000

management.security.enabled=false
```

## TODO 9-04
`registration-server` を再起動してください。

## TODO 9-05
[Hystrix Dashboard](http://localhost:8085/hystrix/monitor?stream=http%3A%2F%2Flocalhost%3A8084%2Fhystrix.stream)を確認してください。

- サーキットブレイカーがOpenになります。
- レイテンシーが1000ミリ秒前後になっています。

## TODO 9-06
`applications/registration-server` の `application.properties` を下記のように変更し、スリープ時間を100ミリ秒に設定してください。

```
spring.application.name=registration-server

server.port=8083

# Change this property
registration.server.latency.ms=100

management.security.enabled=false
```

変更を保存したら、 `registration-server` を再起動してください。

## TODO 9-07
[Hystrix Dashboard](http://localhost:8085/hystrix/monitor?stream=http%3A%2F%2Flocalhost%3A8084%2Fhystrix.stream)を確認してください。

- サーキットブレイカーがClosedに戻ります。

<!------------------------------------------------------------->
# 10. 後片付け
- 全クラスを停止してください。
- 下記のコマンドでコミットしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git add .
git commit -m "completed"
```

> 未コミットのまま次の演習でブランチを切り替えると、新しいブランチにも影響が出てしまいます。必ずコミットを行なってください。

この演習は以上です。