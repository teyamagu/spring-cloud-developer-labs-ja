Lab: Hystrix Stats Aggregation
======================================

# この演習の目標
- Turbineを利用して、複数マイクロサービスからのHystrix Streamを監視します。

# 目標時間
- 45分間

# 1. 準備

## TODO 1-01
下記コマンドで今回の演習用ブランチを作成・切り替えしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git checkout -b my-hystrix-stats-aggregation-start hystrix-stats-aggregation-start
```

<!------------------------------------------------------------->
# 2. マイクロサービスの起動

## TODO 2-01
下記のコマンドで `platform-services/eureka-server` を起動してください。

```bash
./gradlew platform-services:eureka-server:bootRun
```

## TODO 2-02
下記のコマンドで `applications/timesheets-server` を起動してください。

```bash
./gradlew applications:timesheets-server:bootRun
```

## TODO 2-03
下記のコマンドで `applications/registration-server` を起動してください。

```bash
./gradlew applications:registration-server:bootRun
```

## TODO 2-04
ブラウザで[Eureka Server Dashboard](http://localhost:8761)を開いてください。

`registration-server` と `timesheets-server` がそれぞれ1インスタンスずつ登録されていることが分かります。

## TODO 2-05
新規ターミナルウィンドウを開き、下記のコマンドで `timesheets-server` の新規インスタンスをポート番号9084で起動してください。

```bash
SERVER_PORT=9084 ./gradlew applications:timesheet-server:bootRun
```

## TODO 2-06
ブラウザで[Eureka Server Dashboard](http://localhost:8761)を開いてください。

`registration-server` が1インスタンス、 `timesheets-server` 2インスタンス登録されていることが分かります。

<!------------------------------------------------------------->
# Turbineの起動

## TODO 3-01
`platform-services/turbine-server` ディレクトリの `build.gradle` を開き、下記の記述があることを確認してください。

```groovy
apply from: "$projectDir/../server.gradle"

dependencies {
  // TurbineのStarterが含まれている
  compile "org.springframework.cloud:spring-cloud-starter-turbine"
}
```

## TODO 3-02
`spring-cloud-developer-code` ディレクトリの `settings.gradle` を開き、下記の記述があることを確認してください。

```groovy
rootProject.name = "pal-tracker-distributed"

include "applications:registration-server"
include "applications:timesheets-server"

include "components:accounts"
include "components:projects"
include "components:timesheets"
include "components:users"

include "components:rest-support"
include "components:test-support"

// turbine-serverが含まれている
include "platform-services:turbine-server"

include "platform-services:hystrix-dashboard"
include "platform-services:eureka-server"
```

## TODO 3-03
`platform-services/turbine-server` の `TurbineServerApplication` クラスを開き、 `@EnableTurbine` が付加されていることを確認してください。

```java
package io.pivotal.pal.turbineserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.turbine.EnableTurbine;

@SpringBootApplication
@EnableTurbine // Turbineを有効化するアノテーション
public class TurbineServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(TurbineServerApplication.class, args);
	}
}
```

## TODO 3-04
`platform-services/turbine-server` の `application.properties` を開き、下記の設定が記述されていることを確認してください。

```
spring.application.name=turbine-server
server.port=8086

# クラスター名をどこから取得するか
turbine.clusterNameExpression='default'

# Turbineが監視対象とするマイクロサービスのEureka Service ID
turbine.appConfig=TIMESHEETS-SERVER
```

## TODO 3-05
下記のコマンドで `platform-services/turbine-server` を起動してください。

```bash
./gradlew platform-services:turbine-server:bootRun
```

<!------------------------------------------------------------->
# 4. Hystrixデータを集約する

## TODO 4-01
下記のコマンドで、 `timesheets-server` (ポート番号8084)にアクセスしてください。

```bash
while true; sleep .1; do curl -i -XPOST -H"Content-Type: application/json" localhost:8084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}";  done;
```

## TODO 4-02
下記のコマンドで、 `timesheets-server` (ポート番号9084)にアクセスしてください。

```bash
while true; sleep .1; do curl -i -XPOST -H"Content-Type: application/json" localhost:9084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}";  done;
```

## TODO 4-03
ブラウザで http://localhost:8086/turbine.stream にアクセスして、Turbine Streamを見てください。これは、上記2つのインスタンスからのHystrix Streamが1つにまとめられたものです。

見終わったら、このブラウザは閉じてください。

<!------------------------------------------------------------->
# 5. Hystrix Dashboardでの監視

## TODO 5-01
下記のコマンドでHystrix Dashboardを起動してください。

```bash
./gradlew platform-services:hystrix-dashboard:bootRun
```

## TODO 5-02
ブラウザで[Hystrix Dashboard](http://localhost:8085/hystrix)を開いてください。

## TODO 5-03
URL入力欄に「http://localhost:8086/turbine.stream」と入力してください。

> これは「デフォルトクラスター」と呼ばれます。特定のクラスターを指定する場合は「http://localhost:8086/turbine.stream?cluster=クラスター名」と入力します。

## TODO 5-04
[Monitor Stream]ボタンをクリックして監視画面に遷移してください。[Hosts]が「2」になっており、2インスタンスを監視できていることが分かります。

## TODO 5-05
`timesheets-server` のうち片方のインスタンスを停止し（該当ターミナル上で `Ctrl + C` ）、Hystrix Dashboardを確認してください。Circuit Breakerの1つがOpen、もう一方がClosedになります。更に、[Hosts]が「1」になります。

> インスタンス停止 -> Eureka Serverへの反映 -> Turbineへの反映 -> Hystrix Dashboardの反映にはしばらく時間がかかります（1〜2分）。

<!------------------------------------------------------------->
# 6. Turbine Streamの利用

## TODO 6-01
下記のコマンドでRabbit MQを起動してください。

### 直接インストールした場合

```bash
rabbitmq-server
```

### Dockerを使っている場合

```bash
docker container start rabbitmq-scd
```

## TODO 6-02
`turbine-server` と `timesheets-server` の全インスタンスを停止してください。

## TODO 6-03
`applications/timesheets-server` ディレクトリの `build.gradle` を下記のように修正してください。

```groovy
apply from: "$projectDir/../server.gradle"

dependencies {
    // org.springframework.cloud:spring-cloud-starter-hystrix は削除する

    compile project(":components:timesheets")
    
    // 下記2つの依存性を追加
    compile "org.springframework.cloud:spring-cloud-netflix-hystrix-stream"
    compile "org.springframework.cloud:spring-cloud-starter-stream-rabbit"
}
```

## TODO 6-04
`platform-services/turbine-server` ディレクトリの `build.gralde` を下記のように修正してください。

```groovy
apply from: "$projectDir/../server.gradle"

dependencies {
    // org.springframework.cloud:spring-cloud-starter-turbine は削除する

    // 下記2つの依存性を追加
	compile "org.springframework.cloud:spring-cloud-starter-netflix-turbine-stream"
	compile "org.springframework.cloud:spring-cloud-starter-stream-rabbit"
}
```

## TODO 6-05
`platform-services/turbine-server` の `TurbineServerApplication` クラスを下記のように変更してください。

```java
package io.pivotal.pal.turbineserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.turbine.stream.EnableTurbineStream;

// @EnableTurbineアノテーションと該当import文は削除する
@SpringBootApplication
@EnableTurbineStream // このアノテーションを追加
public class TurbineServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(TurbineServerApplication.class, args);
	}
}
```

## TODO 6-06
`turbine-server` と `timesheets-server` の全インスタンスを再起動してください。

## TODO 6-07
ブラウザで[RabbitMQの管理画面](http://localhost:15672/#/)を開き、ユーザー名 `guest` 、 パスワード `guest` でログインしてください。

## TODO 6-08
[Exchangesタブ](http://localhost:15672/#/exchanges)を開いてください。[springCloudHystrixStream]というexchangeが作成されていることが分かります。

> exchangeとは、RabbitMQ内部のキューに対する入り口のようなものです。 `timesheets-server` からは、このexchange名を指定してメッセージ（今回はHystrix Stream）を送信します。

## TODO 6-09
[Queuesタブ](http://localhost:15672/#/queues)を開いてください。[springCloudHystrixStream.anonymous.ランダムな文字列]というキューが作成されていることが分かります。 `timesheets-server` から送信されたHystrix Streamｊは、このキューを通ってTurbineに送られます。

##TODO 6-10
ブラウザで[Hystrix Dashboard](http://localhost:8085/hystrix/monitor?stream=http%3A%2F%2Flocalhost%3A8086%2Fturbine.stream) を開いてください。変わらずCircuit Breakerの状態を監視できていることが分かります。Hystrix Dashboardは、RabbitMQのキューを通してHystrix Streamを取得しています。

<!------------------------------------------------------------->
# 7. 後片付け
- 全クラスおよびRabbitMQを停止してください。
  - Docker利用の場合は `docker container stop rabbitmq-scd` でRabbitMQが停止します。
- 下記のコマンドでコミットしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git add .
git commit -m "completed"
```

> 未コミットのまま次の演習でブランチを切り替えると、新しいブランチにも影響が出てしまいます。必ずコミットを行なってください。

この演習は以上です。