Lab: Spring Cloud Config Server
======================================

# この演習の目標
- Config Serverを作成します。
- Config Clientを作成し、Config Serverから設定を取得します。
- プロファイルによって環境を切り替えます。
- リフレッシュにより、再起動なしで設定を変更します。

# 目標時間
- 45分間

# 1. 準備

## TODO 1-01
下記コマンドで今回の演習用ブランチを作成・切り替えしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git checkout -b my-config-server-start config-server-start
```

<!------------------------------------------------------------->
# 2. Gitリポジトリの作成

## TODO 2-01
下記のコマンドで。ローカルのGitリポジトリを作成してください。

```bash
mkdir -p ~/workspace/config
cd ~/workspace/config
git init
```

> 以下の手順では、このGitリポジトリの絶対パスを `{local-config-directory}` で表します。

## TODO 2-02
`{local-config-directory}` ディレクトリに `application.properties` を作成し、下記のように記述してください。

```
management.security.enabled=false
```

この設定は、全アプリケーションに適用されます。

## TODO 2-03
`applications/timesheets-server` と `applications/registration-server` の `application.properties` から、 `management.security.enabled=false` を削除してください。

## TODO 2-04
`{local-config-directory}` ディレクトリに `timesheets-server.properties` を作成し、下記のように記述してください。

```
registration.server.endpoint=http://localhost:8083
```

## TODO 2-05
`applications/timesheets-server` の `application.properties` から、 `registration.server.endpoint=http://localhost:8083` を削除してください。

## TODO 2-06
下記のコマンドで、設定をコミットしてください。

> このコマンドは `{local-config-directory}` ディレクトリで実行してください。

```bash
git add .
git commit -m"Add registration server endpoint"
```

<!------------------------------------------------------------->
# 3. Config Serverの実装

## TODO 3-01
`platform-services/config-server` ディレクトリを作成してください。このディレクトリ直下に、下記のものを作成してください。

- `src/main/java` ディレクトリ
- `src/main/resources` ディレクトリ
- `build.gradle` (ファイルの内容は下記)

```groovy
apply from: "$projectDir/../server.gradle"

dependencies {
    compile "org.springframework.cloud:spring-cloud-config-server"
}
```

## TODO 3-02
`spring-cloud-developer-code/settings.gradle` を下記のように編集してください。

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
include "platform-services:config-server"
```

## TODO 3-03
`src/main/java` ディレクトリに `io.pal.pivotal.configserver.ConfigServerApp` クラスを下記のように作成してください。

```java
package io.pal.pivotal.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@EnableConfigServer
@SpringBootApplication
public class ConfigServerApp {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApp.class, args);
    }
}
```

## TODO 3-04
`src/main/resources` ディレクトリに `application.properties` を下記の内容で作成してください。

```
server.port=8888
```

## TODO 3-05
 `application.properties` に下記のプロパティを追加してください。

```
spring.cloud.config.server.git.uri={local-config-directory}
```

> `{local-config-directory}` の部分は、先頭に `file:///` を付加したGitリポジトリの絶対パスで置き換えてください。
> macOSでの例: `spring.cloud.config.server.git.uri=file:///Users/user01/workspace/config`

## TODO 3-06
下記のコマンドで `config-server` を起動してください。

```bash
./gradlew platform-services:config-server:bootRun
```

## TODO 3-07
次のコマンドでConfig Serverにアクセスしてください。

```bash
curl localhost:8888/timesheets-server.properties
```

レスポンスの内容が、以前に記述した `timesheets-server.properties` と一致していることを確認してください。

<!------------------------------------------------------------->
# 4. Config Clientの作成

## TODO 4-01
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

    testCompile project(":components:test-support")

    // この依存性を追加
    compile "org.springframework.cloud:spring-cloud-starter-config"
}
```

## TODO 4-02
`timesheets-server` と `registration-server` の
`spring.application.name` プロパティを、 それぞれ `application.properties` から `bootstrap.properties` （ `src/main/resources` に新規作成する）に移してください。

> Config Clientは、起動時に `spring.application.name` で指定された名前でConfig Serverに設定をリクエストしますが、この段階では `application.properties` が読み込まれていません。 `bootstrap.properties` は起動時に読み込まれます。

## TODO 4-03
`timesheets-server` と `registration-server` の `application.properties` から `registration.server.endpoint` プロパティを削除してください。このプロパティはConfig Serverから提供されます。

## TODO 4-04
下記のコマンドで `registration-server` を起動してください。

```bash
./gradlew applications:registration-server:bootRun
```

## TODO 4-05
下記のコマンドで `timesheets-server` を起動してください。

```bash
./gradlew applications:timesheets-server:bootRun
```

## TODO 4-06
下記のコマンドで `timesheets-server` の `/env` Actuatorエンドポイントにアクセスしてください。

```bash
curl localhost:8084/env
```

レスポンスJSONの中に、Config Serverから渡されたプロパティ（ `registration.server.endpoint` と `management.security.enabled` ）が含まれていることを確認してください。

```json
{
  "profiles": [],
  "server.ports": {
    "local.server.port": 8084
  },
  "configService:configClient": {
    "config.client.version": "1ab9227f686a5365d8ac3b0e45508cdfbfa244c0"
  },
  "configService:file:///Users/tada/workspace/config/timesheets-server.properties": {
    "registration.server.endpoint": "http://localhost:8083"
  },
  "configService:file:///Users/tada/workspace/config/application.properties": {
    "management.security.enabled": "false"
  },
  ...
```

## TODO 4-07
下記のコマンドで `timesheets-server` にアクセスし、正常にレスポンスされることを確認してください。

```bash
curl -i -XPOST -H"Content-Type: application/json" localhost:8084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}";
```

> Config ServerのURLは、 Config Clientの `application.properties` に `spring.cloud.config.uri` プロパティ（デフォルト値は `http://localhost:8888` ）で指定します。
> 今はConfig ServerのURLがデフォルト値と同じであるため、特にプロパティを記述しなくてもアクセスできています。

<!------------------------------------------------------------->
# 5. プロファイルの利用

## TODO 5-01
`{local-config-directory}` に `timesheets-server-production.properties` を新規作成し、下記のように記述してください。

```
registration.server.endpoint=http://localhost:10083
```

> これは"production"(本番)環境で `registration-server` がポート番号10083で動いていると仮定しています。

## TODO 5-02
下記のコマンドでコミットしてください。

> このコマンドは `{local-config-directory}` で実行してください。

```bash
git add .
git commit -m "add timesheets-server-production.properties"
```

## TODO 5-03
下記のコマンドで、Config Serverから `timesheets-server-production.properties` の情報を取得してください。

```bash
curl localhost:8888/timesheets-server-production.properties
```

> Config Serverが起動していない場合は、 `./gradlew platform-services:config-server:bootRun` で起動してください。

`timesheets-server-production.properties` に定義したプロパティだけでなく、 `application.properties` に定義されたプロパティも取得できることが分かります。

## TODO 5-04
下記のコマンドで、"production"環境で `registration-server` を起動してください。

```bash
SERVER_PORT=10083 ./gradlew applications:registration-server:bootRun
```

ポート番号を10083に変更して起動しています。

## TODO 5-05
下記のコマンドで、"production"プロファイルで `timesheets-server` を再起動してください。

```bash
SPRING_PROFILES_ACTIVE=production ./gradlew applications:timesheets-server:bootRun
```

## TODO 5-06
下記のコマンドで、 `timesheets-server` の
 `/env` エンドポイントにアクセスしてください。

 ```bash
curl localhost:8084/env
```

レスポンスJSONの中に、Config Serverから渡されたプロパティ（ `registration.server.endpoint` と `management.security.enabled` ）が含まれていることを確認してください。

`registration.server.endpoint` プロパティが2つありますが、今回はプロファイルを指定した `timesheets-server-production.properties` の値が使われます。

```json
{
  "profiles": [
    "production"
  ],
  "server.ports": {
    "local.server.port": 8084
  },
  "configService:configClient": {
    "config.client.version": "928ebaf8a364273f4f656cf7fbd6e12ceee61232"
  },
  "configService:file:///Users/tada/workspace/config/timesheets-server-production.properties": {
    "registration.server.endpoint": "http://localhost:10083"
  },
  "configService:file:///Users/tada/workspace/config/timesheets-server.properties": {
    "registration.server.endpoint": "http://localhost:8083"
  },
  "configService:file:///Users/tada/workspace/config/application.properties": {
    "management.security.enabled": "false"
  },
```

## TODO 5-07
下記のコマンドで、アプリケーションが正しく動作している（レスポンスステータスコードが201になる）ことを確認してください。

```bash
curl -i -XPOST -H"Content-Type: application/json" localhost:8084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}";
```

<!------------------------------------------------------------->
# 6. リフレッシュによる無停止での設定変更

## TODO 6-01
`timesheets-server` の `TimesheetsApp` クラスを下記のように変更して、 `ProjectClient` のBeanを `@RefreshScope` にしてください。

```java
package io.pivotal.pal.tracker.timesheets;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.web.client.RestOperations;

import java.util.TimeZone;


@SpringBootApplication
@ComponentScan({"io.pivotal.pal.tracker.timesheets", "io.pivotal.pal.tracker.restsupport"})
public class TimesheetsApp {

    public static void main(String[] args) {
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
        SpringApplication.run(TimesheetsApp.class, args);
    }

    @RefreshScope // このアノテーションを付加
    @Bean
    ProjectClient projectClient(
        RestOperations restOperations,
        @Value("${registration.server.endpoint}") String registrationEndpoint
    ) {
        return new ProjectClient(restOperations, registrationEndpoint);
    }
}
```

## TODO 6-02
コードの変更を反映するために、 `timesheets-server` を再起動してください。 `Ctrl + C` で停止し、下記のコマンドで起動します。

```bash
SPRING_PROFILES_ACTIVE=production ./gradlew applications:timesheets-server:bootRun
```

## TODO 6-03
下記のコマンドで `timesheets-server` の
 `/env` エンドポイントにアクセスし、 `registration.server.endpoint` の値がまだ10083のままであることを確認してください。

 ```bash
curl localhost:8084/env
```

## TODO 6-04
`{local-config-directory}` ディレクトリの `timesheets-server-production.properties` を下記のように変更してください。

```
registration.server.endpoint=http://localhost:20083
```

## TODO 6-05
下記のコマンドでコミットしてください。

> このコマンドは `{local-config-directory}` で実行してください。

```bash
git add .
git commit -m "change timesheets-server-production.properties"
```

> この時点では、 `/env` エンドポイントにアクセスしても、まだプロパティの値は変更されていません。

## TODO 6-06
下記のコマンドで、 `timesheets-server` をリフレッシュしてください。

```bash
curl -i -XPOST localhost:8084/refresh
```

> この時点で `/env` エンドポイントにアクセスすると、プロパティの値は変更されていることが分かります。

## TODO 6-07
下記のcurlコマンドを実行すると、レスポンスステータスコードが500になります。

```bash
curl -i -XPOST -H"Content-Type: application/json" localhost:8084/time-entries/ -d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}";
```

これは、アクセスする際のURLが変更されたことにより、 `timesheets-server` が `registration-server` に正しくアクセスできなくなったためです。

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