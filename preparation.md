Spring Cloud Developer事前準備手順 (by Casareal, 2019-01-07)
==========================================================

# 研修コンテンツへのアクセス
1. Pivotalアカウントを作成してください。作成手順は[こちら](https://github.com/Pivotal-Japan/cf-workshop/blob/master/pivotal-web-services.md)を参照してください。
2. https://courses.education.pivotal.io/ にアクセスしてください。ユーザー名・パスワードなどは、Pivotalアカウントのものを利用してください。
3. [Spring Cloud]をクリックしてください。
4. [Spring Cloud Introduction]の[LAB]をクリックしてください。
5. [Download the Source Code]の[Download the code repo zip file.]をクリックしてください。これが演習のソースコードです。
6. ダウンロードした `spring-cloud-developer-code.zip` を適当なディレクトリに展開してください。

# インストール

> PCのメモリは8GB以上（可能であれば16GB以上）を推奨します。

## JDK 8
1. Oracle JDK 8を[こちら](https://www.oracle.com/technetwork/java/javase/downloads/index.html)からダウンロード・インストールしてください。
2. 環境変数 `JAVA_HOME` ・ `PATH` を設定してください。

> JDK 9以上は対応していませんので、必ずJDK 8をインストールしてください。

## IDE
1. [IntelliJ IDEA](https://www.jetbrains.com/idea/download/)をインストールしてください。
2. 利用するJDKとして、インストールしたJDK 8を設定してください。
3. 展開した `spring-cloud-developer-code` ディレクトリを、Gradleプロジェクトとしてインポートしてください。

> IntelliJ IDEAはUltimate Editionを推奨しますが、Community Editionでも演習は実施可能です

## Git
### Macの場合
1. Homebrewコマンド `brew install git` でインストールしてください。

### Windowsの場合
1. [Git Bash](https://git-scm.com/downloads)をインストールしてください。

## curl
### Macの場合
1. デフォルトで入っていますのでインストール不要です。

### Windowsの場合
1. Git Bashに含まれていますので別途のインストールは不要です。

## jq
### Macの場合
1. Homebrewコマンド `brew install jq` でインストールしてください。

### Windowsの場合
1. [jqの公式サイト](https://stedolan.github.io/jq/)から `jq-1.6.exe` をダウンロードしてください。
2. ファイル名を `jq.exe` に変更して適当なフォルダに配置し、環境変数PATHに設定してください。

## RabbitMQ
### Macの場合
1. Homebrewコマンド `brew install rabbitmq` でインストールしてください。
2. `sudo rabbitmq-plugins enable rabbitmq_management` コマンドを実行してください。
3. `sudo rabbitmq-plugins enable rabbitmq_tracing` コマンドを実行してください。

### Windowsの場合
1. [Erlang公式サイト](http://www.erlang.org/downloads)から[OTP 21.2 Windows 64-bit Binary File]をダウンロードして、実行してください。
2. [RabbitMQ公式サイト](https://www.rabbitmq.com/install-windows.html)からインストーラーをダウンロードして、実行してください。
3. `rabbitmq-plugins enable rabbitmq_management` コマンドを実行してください。
4. `rabbitmq-plugins enable rabbitmq_tracing` コマンドを実行してください。

### Dockerを利用する場合
1. Dockerを起動してください。
2. `spring-cloud-developer-labs-ja/docker/rabbitmq` ディレクトリ（ `Dockerfile` があります）で `docker image build -t rabbitmq-scd .` コマンドを実行してください。
3. `docker container create -p 5672:5672 -p 15672:15672 --name rabbitmq-scd rabbitmq-scd` コマンドを実行してください。

> 上記のコマンドは `spring-cloud-developer-labs-ja/docker/rabbitmq/Dockerコマンド.txt` にも記載しています。

## JMeter
> この手順は各OS共通です。

1. [JMeterの公式サイト](http://jmeter.apache.org/download_jmeter.cgi?Preferred=ftp%3A%2F%2Fapache.mirrors.tds.net%2Fpub%2Fapache.org%2F)から `apache-jmeter-5.0.zip` をダウンロードしてください。
2. 適当なフォルダにZIPファイルを展開し、 `apache-jmeter-5.0/bin` ディレクトリを環境変数 `PATH` に設定してください。
