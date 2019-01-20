Lab: Fault Tolerance with Hystrix
======================================

# この演習の目標
- サーキットブレイカーパターンを理解します。
- Hystrixによるフォールバックを実装します。

# 1. 準備

## TODO 1-01
下記コマンドで今回の演習用ブランチを作成・切り替えしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git checkout -b my-hystrix-start hystrix-start
```

<!------------------------------------------------------------->
# 2. 初期設定

## TODO 2-01
`applications/timesheets-server/build.gradle` に下記のように記述してください。

```groovy
apply from: "$projectDir/../server.gradle"

dependencies {
    compile project(":components:timesheets")
    // この依存性を追加
    compile "org.springframework.cloud:spring-cloud-starter-hystrix"
}
```

## TODO 2-02
`applications/timesheets-server/src/main/java/io/pivotal/pal/tracker/timesheets/TimesheetsApp.java` にアノテーション `@EnableCircutBreaker` を追加してください。

```java
package io.pivotal.pal.tracker.timesheets;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.web.client.RestOperations;

import java.util.TimeZone;

@EnableCircuitBreaker // このアノテーションを追加
@SpringBootApplication
@ComponentScan({"io.pivotal.pal.tracker.timesheets", "io.pivotal.pal.tracker.restsupport"})
public class TimesheetsApp {

    public static void main(String[] args) {
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
        SpringApplication.run(TimesheetsApp.class, args);
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

<!------------------------------------------------------------->
# 3. クライアントにCircuit Breakerを追加

## TODO 3-01
`components/timesheets/build.gradle` に下記のように記述してください。

```groovy
dependencies {
    compile project(":components:rest-support")
    // この依存性を追加
    compile "com.netflix.hystrix:hystrix-javanica:1.5.12"

    testCompile project(":components:test-support")
}
```

> このライブラリは `@HystrixCommand` アノテーションを提供します。

## TODO 3-02
`components/timesheets/src/main/java/io/pivotal/pal/tracker/timesheets/ProjectClient.java` の `getProject` メソッドにアノテーション `@HystrixCommand(fallbackMethod = "getProjectFromCache")` を追加してください。

## TODO 3-03
`ConcurrentMap` を利用して、読み込んだプロジェクトをインメモリにキャッシュするよう `ProjectClient` クラスを変更してください。

## TODO 3-04
`getProjectFromCache` メソッドを実装してください(戻り値・引数は `getProject` と同じです)。このメソッドでは、ログの出力後、キャッシュからのプロジェクト情報を取得して返してください。

`ProjectClient` クラスの完成形は下記です。

```java
package io.pivotal.pal.tracker.timesheets;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.client.RestOperations;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class ProjectClient {

    private final Logger logger = LoggerFactory.getLogger(ProjectClient.class);

    private final RestOperations restOperations;
    private final String endpoint;
    private final ConcurrentMap<Long, ProjectInfo> cache;

    public ProjectClient(RestOperations restOperations, String registrationServerEndpoint) {
        this.restOperations = restOperations;
        this.endpoint = registrationServerEndpoint;
        cache = new ConcurrentHashMap<>();
    }

    @HystrixCommand(fallbackMethod = "getProjectFromCache")
    public ProjectInfo getProject(long projectId) {
        ProjectInfo info = restOperations.getForObject(endpoint + "/projects/" + projectId, ProjectInfo.class);
        cache.put(projectId, info);
        logger.info("Obtained and cached projectInfo for projectId '{}'", projectId);
        return info;
    }

    public ProjectInfo getProjectFromCache(long projectId) {
        logger.info("Fell back to using cached project lookup for projectId '{}'", projectId);
        return cache.get(projectId);
    }
}
```

## TODO 3-05
`applications/registration-server` の `RegistrationApp` クラスおよび`applications/timesheets-server` の `TimesheetsApp` クラスを起動してください。

## TODO 3-06
動作確認のため、下記コマンドを実行してください。

```bash
curl -i -XPOST -H"Content-Type: application/json" localhost:8084/time-entries/ \
-d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}"
```

コンソールに `getProject` メソッドのログが出力されていることを確認してください。

## TODO 3-07
`applications/registration-server` の `RegistrationApp` クラスを停止してください。

## TODO 3-08
同じcurlコマンドを実行してください。201レスポンスが返ってきますが、コンソールを見るとフォールバックメソッド `getProjectFromCache` が実行されていることが分かります。

<!------------------------------------------------------------->
# 4. Hystrix Streamの確認

## TODO 4-01
ブラウザで http://localhost:8084/hystrix.stream を開いてください。

## TODO 4-02
内容を確認したらブラウザを閉じ、全クラスを停止してください。

<!------------------------------------------------------------->
# 5. Hystrix Dashboardの作成

## TODO 5-01
`platform-services` に `hystrix-dashboard` ディレクトリを作成し、その直下に `build.gradle` を下記のように作成してください。

```groovy
apply from: "$projectDir/../server.gradle"

dependencies {
    // Hystrix DashboardのStarter
    compile "org.springframework.cloud:spring-cloud-starter-hystrix-dashboard"
}
```

## TODO 5-02
`spring-cloud-developer-code/settings.gradle` を下記のように記述してください。

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

// この行を追加
include "platform-services:hystrix-dashboard"
```

## TODO 5-03
`platform-services/hystrix-dashboard` ディレクトリに `src/main/java` ディレクトリを作成してください。

## TODO 5-04
`src/main/java` ディレクトリに `io.pivotal.pal.tracker.hystrixdashboard.HystrixDashboardApp` クラスを下記のように作成してください。

```java
package io.pivotal.pal.tracker.hystrixdashboard;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;

@EnableHystrixDashboard // Hystrix Dashboardを有効化する
@SpringBootApplication
public class HystrixDashboardApp {

    public static void main(String[] args) {
        SpringApplication.run(HystrixDashboardApp.class, args);
    }
}
```

## TODO 5-05
`platform-services/hystrix-dashboard` ディレクトリに `src/main/resources` ディレクトリを作成し、その直下に `application.properties` を下記のように作成してください。

```
spring.application.name=hystrix-dashboard
server.port=8085
```

## TODO 5-06
`platform-services/hystrix-dashboard` の `HystrixDashboardApp` クラスを起動してください。

## TODO 5-07
`applications/registration-server` の `RegistrationApp` クラスを起動してください。

## TODO 5-08
`applications/timesheets-server` の `TimesheetsApp` クラスを起動してください。

## TODO 5-09
ブラウザで http://localhost:8085/hystrix を開いてください。これがHystrix Dashboardです。

## TODO 5-10
Hystrix DashboardのURL入力欄に `http://localhost:8084/hystrix.stream` と入力し[Monitor Stream]ボタンをクリックしてください。

## TODO 5-11
`timesheets-server` に負荷をかけるために、下記のコマンドを実行してください。0.1秒ごとに1回、curlコマンドを実行しています。

```bash
while true; sleep .1; do curl -i -XPOST -H"Content-Type: application/json" \
localhost:8084/time-entries/ \
-d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}";  done;
```

## TODO 5-12
`components/projects` の `ProjectController#get` メソッドでは、レスポンスを意図的に遅くするための処理が記述されています。

- 30秒間はすぐにレスポンスを返します。サーキットはClosedのままです。
- 各1分間の後半30秒は、レスポンスに7秒かかります。これによりタイムアウト（デフォルトの閾値が5秒間であるため）が発生します。
  - `requestVolumeThreshold` を `2` に指定すると、サーキットがOpenになります。

During a half minute, the controller will respond promptly, and the circuit will be closed.
In the second half of each minute, a 7-second latency in the response is introduced, which will cause the circuit to timeout after 5 seconds. Combined with the requestVolumeThreshold setting of 2 (two failures is sufficient to trip the circuit in a given time window), you will see the circuit open.

<!------------------------------------------------------------->
# 6. インスタンスダウン障害のハンドリング

## TODO 6-01
`applications/registration-server` の `RegistrationApp` クラスを停止してください。サーキットがOpenになります。

## TODO 6-02
`applications/registration-server` の `RegistrationApp` クラスを再起動してください。しばらく時間が経つとサーキットがClosedになります。

<!------------------------------------------------------------->
# 7. 後片付け
- 全クラスを停止してください。
- 下記のコマンドでコミットしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git add .
git commit -m "completed"
```

> 未コミットのまま次の演習でブランチを切り替えると、新しいブランチにも影響が出てしまいます。必ずコミットを行なってください。

この演習は以上です。