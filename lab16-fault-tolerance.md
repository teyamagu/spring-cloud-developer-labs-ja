Lab: Fault Tolerance
======================================

# この演習の目標
- Config Serverに障害があった際のConfig Clientの振る舞いを確認します。
- Config Clientにリトライなどの設定を行います。

# 目標時間
- 30分間

# 1. 準備

## TODO 1-01
下記コマンドで今回の演習用ブランチを作成・切り替えしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git checkout -b my-config-server-fault-start config-server-fault-start
```

<!------------------------------------------------------------->
# 2. リポジトリの準備

> この手順は、[Spring Cloud Config Serverの演習](lab13-spring-cloud-config-server.md)の演習を完了している方は飛ばしてください。

## TODO 2-01
下記のコマンドでディレクトリの作成・移動を行なってください。

### macOS/Linuxの場合

```bash
mkdir -p ${HOME}/workspace/config
cd ${HOME}/workspace/config
```

### Windowsの場合

```bash
cd %HOME%
mkdir workspace
mkdir config
cd %HOME%\workspace\config
```

## TODO 2-02
下記のコマンドでGitリポジトリを初期化してください。

```bash
git init
```

## TODO 2-03
`application.properties` を下記の内容で新規作成してください。

```
management.security.enabled=false
```

## TODO 2-04
`timesheets-server.properties` を下記の内容で新規作成してください。

```
registration.server.endpoint=http://localhost:8083
```

## TODO 2-05
下記のコマンドでコミットしてください。

```bash
git add application.properties timesheets-server.properties
git commit -m"Add registration server endpoint"
```

<!------------------------------------------------------------->
# 3. Config Clientのキャッシュ

## TODO 3-01
下記のコマンドで `config-server` を起動してください。

```bash
./gradlew platform-services:config-server:bootRun
```

## TODO 3-02
下記のコマンドで `registration-server` と `timesheets-server ` を起動してください。

```bash
./gradlew applications:registration-server:bootRun
```

```bash
./gradlew applications:timesheets-server:bootRun
```

## TODO 3-03
下記のコマンドで正しく実行できることを確認してください。

```bash
curl -X POST -H "Content-Type: application/json" localhost:8084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}";
```

## TODO 3-04
`config-server` を停止してください。

## TODO 3-05
下記のコマンドで正しく実行できることを確認してください。

```bash
curl -X POST -H "Content-Type: application/json" localhost:8084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}";
```

一旦Config Serverから受け取った設定はConfig Clientでキャッシュされています。なので、Config Serverが停止してもConfig Clientは正しく動作します。

<!------------------------------------------------------------->
# 4. ファストフェイル

## TODO 4-01
`applications/timesheets-server` の `build.gradle` を下記のように編集してください。

```groovy
apply from: "$projectDir/../server.gradle"

dependencies {
    compile project(":components:timesheets")

    // この2つの依存性を追加
    compile "org.springframework.retry:spring-retry"
    compile "org.springframework.boot:spring-boot-starter-aop"
}
```

## TODO 4-02
`applications/timesheets-server` の `bootstrap.properties` を下記のように編集してください。

```
spring.application.name=timesheets-server

# Add this property
spring.cloud.config.fail-fast=true
```

> `spring.cloud.config.fail-fast=true` は、Config Serverへの接続に失敗すると起動しないという設定です。デフォルトは `false` で、失敗時はローカルの `application.properties` などの値を利用して起動を試みます。

## TODO 4-03
`timesheets-server` を再起動してください（ `config-server` は起動しないでください）。

コンソールのログを確認してください。何回かリトライしたものの `config-server` から設定を取得できないため、起動に失敗します。

<!------------------------------------------------------------->
# 5. 障害時の一時的リカバリ

## TODO 5-01
`applications/timesheets-server` の `bootstrap.properties` を下記のように編集してください。

```
spring.application.name=timesheets-server

spring.cloud.config.fail-fast=true

# Add these properties
spring.cloud.config.retry.initial-interval=2000
spring.cloud.config.retry.max-interval=30000
spring.cloud.config.retry.multiplier=1.5
```

| プロパティ名 | 説明 | デフォルト値 |
|------------|-----|------------|
| `initital-interval` | 1回目に失敗した際、2回目を行うまでの時間間隔をミリ秒で指定します。 | `1000` |
| `multiplier` | N個目の時間間隔（N回目とN+1回目の間）に `multiplier` を掛けた値が、N+1個目の時間間隔（N+1回目とN+2回目の間）になります。 | `1.1` |
| `max-interval` | 最大の時間間隔（ミリ秒）です。何回リトライを行なっても、時間間隔はこの値を超えることはありません。 | `2000` |
| `max-attempts` | 処理を行う最大回数です。この回数に達しても処理が失敗する場合、例外が発生してリトライを終了します。 | `6` |

## TODO 5-02
`timesheets-server` を再起動してください。

## TODO 5-03
リトライが終了する前に `config-server` を再起動してください。再起動が完了後、 `timesheets-server` のコンソールログを確認してください。 `timesheets-server` は正しく再起動されます。

## TODO 5-04
下記のコマンドで正しく実行できることを確認してください。

```bash
curl -X POST -H "Content-Type: application/json" localhost:8084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}";
```

<!------------------------------------------------------------->
# 6. 後片付け
- 全クラスを停止してください。
- 下記のコマンドでコミットしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git add .
git commit -m "completed"
```

> 未コミットのまま次の演習でブランチを切り替えると、新しいブランチにも影響が出てしまいます。必ずコミットを行なってください。

この演習は以上です。