Lab: Service Discovery Client
======================================

# この演習の目標
- Eureka Clientを作成・実行します。
- Eureka ServerとEureka Clientを連携して動作させます。
- Eurekaを利用して、別のEureka Clientにアクセスします。

# 1. 準備

## TODO 1-01
下記コマンドで今回の演習用ブランチを作成・切り替えしてください。

> このコマンドは演習のルート `spring-cloud-developer-code` ディレクトリで実行してください。

```bash
git checkout -b my-discovery-client-start discovery-client-start
git cherry-pick service-registry-build-eureka-server
```

<!------------------------------------------------------------->
# 2. Starterの追加

## TODO 2-01
`applications/server.gradle` に下記のように記述してください。

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
    compile "org.springframework.cloud:spring-cloud-starter-netflix-eureka-client"
}
```

<!------------------------------------------------------------->
# 3. Eureka Clientの有効化
`@EnableEurekaClient` を付加することでEureka Clientを有効化します。

## TODO 3-01
`applications/registration-server/src/main/java/io/pivotal/pal/tracker/registration/RegistrationApp.java`

```java
package io.pivotal.pal.tracker.registration;

import io.pivotal.pal.tracker.projects.data.ProjectDataGateway;
import io.pivotal.pal.tracker.projects.data.ProjectFields;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.context.annotation.ComponentScan;

import javax.annotation.PostConstruct;
import java.util.TimeZone;

import static io.pivotal.pal.tracker.projects.data.ProjectFields.projectFieldsBuilder;


@EnableEurekaClient // このアノテーションを付加
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

}
```

## TODO 3-02
`applications/timesheets-server/src/main/java/io/pivotal/pal/tracker/timesheets/TimesheetsApp.java`

```java
package io.pivotal.pal.tracker.timesheets;

import io.pivotal.pal.tracker.restsupport.ServiceLocator;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.web.client.RestOperations;

import java.util.TimeZone;

@EnableEurekaClient // このアノテーションを付加
@SpringBootApplication
@ComponentScan({"io.pivotal.pal.tracker.timesheets", "io.pivotal.pal.tracker.restsupport"})
public class TimesheetsApp {

    public static void main(String[] args) {
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
        SpringApplication.run(TimesheetsApp.class, args);
    }

    @Bean
    ProjectClient projectClient(
            ServiceLocator serviceLocator,
            RestOperations restOperations

    ) {
        return new ProjectClient(serviceLocator,restOperations);
    }
}
```

<!------------------------------------------------------------->
# 4. Service Locatorの作成

## TODO 4-01
`components/rest-support/build.gradle` を下記のように記述してください。

```groovy
dependencies {
    compile "com.fasterxml.jackson.core:jackson-core:$jacksonVersion"
    compile "com.fasterxml.jackson.core:jackson-databind:$jacksonVersion"
    compile "com.fasterxml.jackson.core:jackson-annotations:$jacksonVersion"

    compile "org.springframework:spring-web:$springVersion"
    
    // 下記の依存性を追加
    compile "org.springframework.cloud:spring-cloud-commons:1.3.2.RELEASE"
    compile "org.springframework.cloud:spring-cloud-starter-netflix-eureka-client:1.4.3.RELEASE"
}
```

## TODO 4-02
`components/rest-support/src/main/java` に、`io.pivotal.pal.tracker.restsupport.ServiceLocator` クラスを下記のように作成してください。

```java
package io.pivotal.pal.tracker.restsupport;

public interface ServiceLocator {
    String lookUpServiceUrl(String serviceName);
}
```

## TODO 4-03
`components/rest-support/src/main/java` に、`io.pivotal.pal.tracker.restsupport.EurekaServiceLocator` クラスを下記のように作成してください。

```java
package io.pivotal.pal.tracker.restsupport;

import com.netflix.discovery.EurekaClient;

public class EurekaServiceLocator implements ServiceLocator {
    private EurekaClient eurekaClient;

    public EurekaServiceLocator(EurekaClient eurekaClient) {
        this.eurekaClient = eurekaClient;
    }

    public String lookUpServiceUrl(String serviceName) {
        return eurekaClient.getNextServerFromEureka(serviceName,
                false).getHomePageUrl();
    }
}
```

## TODO 4-04
`components/rest-support/src/main/java` の`io.pivotal.pal.tracker.restsupport.RestConfig` クラスを下記のように編集してください。

```java
package io.pivotal.pal.tracker.restsupport;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.netflix.discovery.EurekaClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestOperations;
import org.springframework.web.client.RestTemplate;


@Configuration
public class RestConfig {

    //--------------------------------------------- ここから
    private final EurekaClient eurekaClient;

    public RestConfig(EurekaClient eurekaClient) {
        this.eurekaClient = eurekaClient;
    }

    @Bean
    ServiceLocator getServiceLocator() {
        return new EurekaServiceLocator(eurekaClient);
    }
    //---------------------------------------- ここまでを追加

    @Bean
    public RestOperations restOperations() {
        return new RestTemplate();
    }

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return mapper;
    }
}
```

## TODO 4-05
`components/timesheets/src/main/java` の `io.pivotal.pal.tracker.timesheets.ProjectClient` クラスを下記のように編集してください。

```java
package io.pivotal.pal.tracker.timesheets;

import io.pivotal.pal.tracker.restsupport.ServiceLocator;
import org.springframework.web.client.RestOperations;

public class ProjectClient {

    private final ServiceLocator serviceLocator;
    private final RestOperations restOperations;

    public ProjectClient(ServiceLocator serviceLocator,
                         RestOperations restOperations) {
        this.serviceLocator = serviceLocator;
        this.restOperations = restOperations;
    }

    public ProjectInfo getProject(long projectId) {
        // Eureka ClientでIPアドレスを解決する
        String endpoint = serviceLocator.lookUpServiceUrl("registration-server");
        // そのIPアドレスでアクセスする
        return restOperations.getForObject(endpoint + "/projects/"
                + projectId, ProjectInfo.class);
    }
}
```

## TODO 4-06
`components/timesheets/src/main/java` の `io.pivotal.pal.tracker.timesheets.TimesheetsApp` クラスを下記のように編集してください。

```java
package io.pivotal.pal.tracker.timesheets;

import io.pivotal.pal.tracker.restsupport.ServiceLocator;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.web.client.RestOperations;

import java.util.TimeZone;

@EnableEurekaClient
@SpringBootApplication
@ComponentScan({"io.pivotal.pal.tracker.timesheets", "io.pivotal.pal.tracker.restsupport"})
public class TimesheetsApp {

    public static void main(String[] args) {
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
        SpringApplication.run(TimesheetsApp.class, args);
    }

    @Bean
    ProjectClient projectClient(
            ServiceLocator serviceLocator,
            RestOperations restOperations

    ) {
        // Service Locatorを使うよう変更
        return new ProjectClient(serviceLocator,restOperations);
    }
}
```

<!------------------------------------------------------------->
# 5. 実行

## TODO 5-01
`platform-services/eureka-server` の `EurekaServerApp` クラスを起動してください。

## TODO 5-02
`applications/registration-server` の `RegistrationApp` クラスを起動してください。

## TODO 5-03
`applications/timesheets-server` の `TimesheetsApp` クラスを起動してください。

## TODO 5-04
[Eureka Serverダッシュボード](http://localhost:8761/)をブラウザで開き、 `registration-server` と `timesheets-server` が登録されていることを確認してください。

> `registration-server` や `timesheets-server` などのEureka Serverに登録されている名前は、各アプリケーションの `application.properties` に `spring.application.name` として設定されているものです。

## TODO 5-05
`timesheets-server` が正しく動作しているか、下記のcurlコマンドで確認してください。1つ目のコマンドで登録したデータが、2つ目のコマンドで取得できれば成功です。

```bash
curl -X POST -H"Content-Type: application/json" localhost:8084/time-entries/ \
-d"{\"projectId\": 1, \"userId\": 1, \"date\": \"2015-05-17\", \"hours\": 6}" \
curl localhost:8084/time-entries?userId=1
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