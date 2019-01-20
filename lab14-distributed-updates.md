Lab: Distributed Updates
======================================

# この演習の目標
- Spring Cloud Bus + RabbitMQを利用して、複数インスタンスの設定を一括で更新します。

# 目標時間
- 30分間

# 1. 準備

## TODO 1-01
下記コマンドで今回の演習用ブランチを作成・切り替えしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git checkout -b my-config-server-dist-update-start config-server-dist-update-start
```

<!------------------------------------------------------------->
# 2. Spring Cloud Busの依存性追加

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
    compile "org.springframework.cloud:spring-cloud-starter-config"

    testCompile project(":components:test-support")

    // この依存性を追加
    compile "org.springframework.cloud:spring-cloud-starter-bus-amqp"
}
```

<!------------------------------------------------------------->
# 3. RabbitMQの起動

## TODO 3-01
下記のコマンドでRabbit MQを起動してください。

### 直接インストールした場合

```bash
rabbitmq-server
```

### Dockerを使っている場合

```bash
docker container start rabbitmq-scd
```

## TODO 3-02
ブラウザで[RabbitMQの管理画面](http://localhost:15672/)を開き、ユーザー名 `guest` 、 パスワード `guest` でログインしてください。

<!------------------------------------------------------------->
# 4. マイクロサービスの起動

## TODO 4-01
下記のコマンドで `platform-services/config-server` 、 `applications/registration-server` 、 `applications/timesheets-server` を起動してください。

```bash
./gradlew platform-services:config-server:bootRun
```

```bash
./gradlew applications:registration-server:bootRun
```

```bash
./gradlew applications:timesheets-server:bootRun
```

## TODO 4-02
[RabbitMQの管理画面](http://localhost:15672/)で[Exchangesタブ](http://localhost:15672/#/exchanges)を開くと、[springCloudBus]というexchangeが作成されていることが分かります。また、[Queuesタブ](http://localhost:15672/#/queues)を開くと、[springCloudBus.anonymous.ランダムな文字列]というキューが2つ作成（アプリケーションインスタンスの数だけ）されていることが分かります。

## TODO 4-03
下記のコマンドで、 `timesheets-server` のインスタンスをポート番号8184で起動してください。

```bash
SERVER_PORT=8184 ./gradlew applications:timesheets-server:bootRun
```

## TODO 4-04
`{local-config-directory}` 内の `timesheets-server.properties` を下記のように編集してください。

```
registration.server.endpoint=http://localhost:8083

# Add this property
logging.level.com.netflix=INFO
```

編集が完了したら、下記のコマンドでコミットしてください。

> このコマンドは `{local-config-directory}` で実行してください。

```bash
git add .
git commit -m "Add logging property"
```

## TODO 4-05
下記のコマンドで、ポート番号8084の `timesheets-server` をリフレッシュしてください。

```bash
curl -XPOST localhost:8084/refresh
```

## TODO 4-06
下記のコマンドで、ポート番号8084の `timesheets-server` の `/env` エンドポイントにアクセスしてください。 `logging.level.com.netflix=INFO` プロパティが含まれていることが分かります。

```bash
curl localhost:8084/env
```

次に、下記のコマンドで、ポート番号8184の `timesheets-server` の `/env` エンドポイントにアクセスしてください。 `logging.level.com.netflix=INFO` プロパティは含まれていません。こちらのインスタンスは、リフレッシュがされていないからです。

```bash
curl localhost:8184/env
```

<!------------------------------------------------------------->
# 5. Spring Cloud Busによる一括更新

## TODO 5-01
`{local-config-directory}` 内の `timesheets-server.properties` を下記のように編集してください。

```
registration.server.endpoint=http://localhost:8083

# Change this property
logging.level.com.netflix=DEBUG
```

編集できたら、下記のコマンドでコミットしてください。

> このコマンドは `{local-config-directory}` で実行してください。

```bash
git add .
git commit -m "Change logging level"
```

## TODO 5-02
RabbitMQの管理画面で[Traceタブ](http://localhost:15672/#/traces)を開いてください。[Add a new trace]-[Name]に `SpringCloudBus` という名前を付け、[Add trace]ボタンをクリックしてください。

## TODO 5-03
下記のコマンドで、ポート番号8084の `timesheets-server` の `/bus/refresh` エンドポイントにアクセスしてください。 

```bash
curl -X POST -H "Content-Type: application/json" http://localhost:8084/bus/refresh
```

## TODO 5-04
`timesheets-server` のコンソールでログを確認してください。Config Serverからの設定取得、RabbitMQからのメッセージ取得などが確認できます。

## TODO 5-05
ブラウザで[RabbitMQのTraceログ](http://localhost:15672/api/trace-files/SpringCloudBus.log)を確認してください。

3つのキューでメッセージが送受信されていることが分かります。 `timesheets-server` 2つと `registration-server` 1つで計3つです。今回は `timesheets-server` のみを更新したいのに、 `registration-server` まで更新されています。この点は、のちほど修正します。

## TODO 5-06
下記のコマンドで、 `timesheets-server` 2つのインスタンスの `/env` エンドポイントにアクセスしてください。共に `logging.level.com.netflix=DEBUG` プロパティが含まれていることが分かります。

```bash
curl localhost:8084/env
```

```bash
curl localhost:8184/env
```

## TODO 5-07
[Currently running traces]-[SpringCloudBus]の[Stop]ボタンと、[Trace log files]-[SpringCloudBus.log]の[Delete]ボタンをクリックしてください。これにより、トレースが削除されます。

<!------------------------------------------------------------->
# 6. リフレッシュのフィルタリング

## TODO 6-01
`{local-config-directory}` 内の `timesheets-server.properties` を下記のように編集してください。

```
registration.server.endpoint=http://localhost:8083

# Back to INFO level
logging.level.com.netflix=INFO
```

## TODO 6-02
`{local-config-directory}` 内に `registration-server.properties` を下記のように作成してください。

```
logging.level.com.netflix=DEBUG
```

編集できたら、下記のコマンドでコミットしてください。

> このコマンドは `{local-config-directory}` で実行してください。

```bash
git add .
git commit -m "Change logging level"
```

## TODO 6-03
RabbitMQの管理画面で[Traceタブ](http://localhost:15672/#/traces)を開いてください。[Add a new trace]-[Name]に `SpringCloudBus` という名前を付け、[Add trace]ボタンをクリックしてください。

## TODO 6-04
下記のコマンドで、 `timesheets-server` のみリフレッシュを行ってください。

```bash
curl -X POST -H "Content-Type: application/json" \
http://localhost:8084/bus/refresh?destination=timesheets-server:**
```

> `/bus/refresh?destination=アプリケーション名:ポート番号` で、指定したアプリケーション( `spring.application.name` )・ポート番号のインスタンスのみを更新します。ポート番号を `**` とすると、そのアプリケーションの全インスタンスが更新されます。

## TODO 6-05
`timesheets-server` のコンソールでログを確認してください。Config Serverからの設定取得、RabbitMQからのメッセージ取得などが確認できます。

## TODO 6-06
ブラウザで[RabbitMQのTraceログ](http://localhost:15672/api/trace-files/SpringCloudBus.log)を確認してください。

2つのキューのみでメッセージが送受信されていることが分かります。

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