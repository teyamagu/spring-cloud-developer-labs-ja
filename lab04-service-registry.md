Lab: Service Registry
======================================

# この演習の目標
- Eureka Serverを作成・実行します。
- Eureka Serverに必要な設定を学習します。

# 1. 準備

## TODO 1-01
下記コマンドで今回の演習用ブランチを作成・切り替えしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git checkout -b my-eureka-server-start eureka-server-start
```

<!------------------------------------------------------------->
# 2. Eureka Serverの実装

## TODO 2-01
1. `platform-serivces` ディレクトリ直下に `eureka-server` ディレクトリを作成してください。
2. `eureka-server` ディレクトリ直下に `src/main/java` ・ `src/main/resources` ディレクトリを作成してください。
3. `eureka-server` ディレクトリ直下に `build.gradle` ファイルを作成し、下記のように記述してください。

```groovy
apply from: "$projectDir/../server.gradle"

dependencies {
    compile "org.springframework.cloud:spring-cloud-starter-eureka-server"
}
```

## TODO 2-02
`spring-cloud-developer-code/settings.gradle` に下記の記述を追加してください。

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
include "platform-services:eureka-server"
```

## TODO 2-03
`eureka-server/src/main/java` ディレクトリ直下に `io.pivotal.pal.tracker.eurekaserver.EurekaServerApp` クラスを作成し、下記のように記述してください。

```java
package io.pivotal.pal.tracker.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApp {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApp.class, args);
    }
}
```

## TODO 2-04
`eureka-server/src/main/resources` ディレクトリ直下に `application.yml` ファイルを作成し、下記のように記述してください。

```yaml
spring:
  application:
    name: eureka-server

server:
  port: 8761

management:
  security:
    enabled: false

logging:
  level:
    com.netflix: DEBUG
```

## TODO 2-05
`EurekaServerApp` クラスを起動してください。すると、下記のような例外スタックトレースが出力されます。

```bash
2019-01-08 13:17:34.214 DEBUG 11023 --- [           main] n.d.s.t.j.AbstractJerseyEurekaHttpClient : Jersey HTTP GET http://localhost:8761/eureka//apps/?; statusCode=N/A
2019-01-08 13:17:34.219 ERROR 11023 --- [           main] c.n.d.s.t.d.RedirectingEurekaHttpClient  : Request execution error

com.sun.jersey.api.client.ClientHandlerException: java.net.ConnectException: Connection refused (Connection refused)
	at com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:187) ~[jersey-apache-client4-1.19.1.jar:1.19.1]
	at com.sun.jersey.api.client.filter.GZIPContentEncodingFilter.handle(GZIPContentEncodingFilter.java:123) ~[jersey-client-1.19.1.jar:1.19.1]
	at com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27) ~[eureka-client-1.7.0.jar:1.7.0]
    ...
```

原因は、スタックトレース直前のDEBUGログで分かります。`http://localhost:8761/eureka//apps/?` にアクセスしようとしていますが、このようなURLは存在しないため例外が発生しています。この例外は、後ほど修正します。

## TODO 2-06
[Eureka Serverのダッシュボード](http://localhost:8761)にブラウザでアクセスしてください。

[Instances currently registered with Eureka]の欄を見ると、Eureka Server自身も登録されていることが分かります。この点は、後ほど修正します。

## TODO 2-07
[Eureka Serverに登録されたアプリケーションを確認できるエンドポイント](http://localhost:8761/eureka/apps)をブラウザで確認してください。

今はEureka Server自身のみの情報が確認できます。

<!------------------------------------------------------------->
# 3. Eureka Serverの設定

## TODO 3-01
`eureka-server/src/main/resources/application.yml` を下記のように記述してください。

```yaml
spring:
  application:
    name: eureka-server

server:
  port: 8761

management:
  security:
    enabled: false

logging:
  level:
    com.netflix: DEBUG

# Add properties below
eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

## TODO 3-02
`EurekaServerApp` クラスを再起動してください。URLが適切に設定されているため、先ほど発生した例外のスタックトレースが無いことが分かります。

## TODO 3-03
[Eureka Serverのダッシュボード](http://localhost:8761)にブラウザでアクセスしてください。

[Instances currently registered with Eureka]の欄を見ると、Eureka Server自身は登録されておらず、何も表示されていません。

## TODO 3-04
[Eureka Serverに登録されたアプリケーションを確認できるエンドポイント](http://localhost:8761/eureka/apps)をブラウザで確認してください。

アプリケーションが何も登録されていないことが分かります。

<!------------------------------------------------------------->
# 4. 後片付け
- `EurekaServerApp` クラスを停止してください。
- 下記のコマンドでコミットしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git add .
git commit -m "completed"
```

> 未コミットのまま次の演習でブランチを切り替えると、新しいブランチにも影響が出てしまいます。必ずコミットを行なってください。

この演習は以上です。