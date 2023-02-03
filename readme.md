### EC2にelasticsearchの環境を構築
#### ホストPCからEC2に接続(SSH)
```
ssh -i 鍵 ec2-user@パブリックIP
```

#### 使用コマンド
* サービスの起動状態を確認
```
sudo systemctl | grep elastic
```
サービスの起動状態を確認したいだけならgrepによる絞り込みは不要。

#### 不明点
* yum updateを最初に行う理由が不明。何もインストールしていなければ必要ないのでは?→最初からインストールされているものがある。

* GPG(GNU Privacy Guard)...暗号化ソフトウェア。インストーラーが本物であるかどうか検証するための仕組み。
```
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
**ここで結局何をしているのか。このコマンドはyumではできない?** →鍵のインポートをしている。

* systemd...コンピュータのシステムを起動するときに様々なプログラムを動かす、元のプログラムのこと。  
→systemdに追加することで起動時にプログラムを動かしてくれる。

* discovery.type=single-nodeの意味...クラスタの構成を決めるオプション。単一ノードか複数ノードか。
  * クラスタ...複数サーバーで役割を分担して一つのサービスとしてふるまう構成。
  * ノード...

* **daemon-reload**...enableとセットで使用されているように見える。どのような命令か不明。

* rpm...linuxのパッケージ管理コマンド。yumと似ている。**違いは?**
  * rpm...パッケージ個々の管理をするコマンド。依存関係を考慮しない。パッケージ個々情報表示がyumより詳細である。
  * yum...パッケージの依存関係を考慮してインストールしてくれる。**基本yumでよい?rpmを使用するタイミングは?**

  * rpm...オフラインでのインストール。curlでGETしたものをオフラインでインストールしている。(YUMの過程にある鍵のインポートのみ例外)
  * yum...オンラインでのインストール(repoファイルに記載したURLからインストール)

YUMの手順
1. 公開鍵を取得。
2. URLを辿ってインストーラー取得。
3. 公開鍵を使用してハッシュ値を比較。正しければインストールされる。

* rpmによる情報取得
  ```
  Name        : kibana
  Version     : 7.17.8
  Release     : 1
  Architecture: x86_64
  Install Date: Mon 30 Jan 2023 03:46:34 PM JST
  Group       : default
  Size        : 691236670
  License     : Elastic License
  Signature   : RSA/SHA512, Sat 03 Dec 2022 12:33:10 PM JST, Key ID d27d666cd88e42b4
  Source RPM  : kibana-7.17.8-1.src.rpm
  Build Date  : Fri 02 Dec 2022 09:30:46 PM JST
  Build Host  : kb-c2-16-5deab18561b964d1.c.elastic-kibana-ci.internal
  Relocations : /
  Packager    : Kibana Team <info@elastic.co>
  Vendor      : Elasticsearch, Inc.
  URL         : https://www.elastic.co
  Summary     : Explore and visualize your Elasticsearch data
  Description :
  Explore and visualize your Elasticsearch data
  ```
* yumによる情報取得
  ```
  Loaded plugins: extras_suggestions, langpacks, priorities, update-motd
  Installed Packages
  Name        : kibana
  Arch        : x86_64
  Version     : 7.17.8
  Release     : 1
  Size        : 659 M
  Repo        : installed
  From repo   : kibana-7.x
  Summary     : Explore and visualize your Elasticsearch data
  URL         : https://www.elastic.co
  License     : Elastic License
  Description : Explore and visualize your Elasticsearch data
  ```

#### 環境構築の流れ
1. 必要なソフトをインストール(java8)。リポジトリの更新も。
2. 外部アクセスの設定を行う。(localhost以外からのアクセスを許可)
3. システム起動時にelasticsearch/kibanaが立ち上がるように設定→手動で毎回立ち上げるのであれば書かなくてもよい。
4. **外部からのアクセスを許可する。** →セキュリティグループでの許可が必要  
CentOS上で上記の実装をする場合、
```
sudo firewall-cmd --add-port=9200/TCP --zone=public --permanent
success
```
**この実装がAWSのセキュリティグループの代わりになっている?** →そう。AWS上のサービスで管理できるのでamazonは推奨している。


参考資料  
yumインストール  
https://www.elastic.co/guide/en/beats/filebeat/7.17/setup-repositories.html

docker-composeでの作成に必要なもの
* 設定ファイルに参照先を記入。(マウント先)
```
- type: log
  paths:
    - /var/log/testlog/*.log
```
上記の例はテンプレート「Log」を分解したもの。(docker-composeでVolumeを編集すればよさそう)  
参考資料  
https://www.elastic.co/guide/en/beats/filebeat/7.17/filebeat-input-log.html

* Elasticsearch OutPutの指定  
localhostでは参照できないのでdocker-composeのサービス名(名前をつけるとしたらelasticsearchかdbかな)

* Loggingの表示するログの設定  
Logging.levelをdebugにして動作確認する

docker-composeでの作成に必要なコマンド
設定ファイル編集(filebeat場所は同じかわからん)
```
sudo nano /etc/filebeat/filebest.yml
```
システム再起動
```
sudo systemctl restart filebeat
```
システム状況
```
sudo systemctl status filebeat
```
ファイルの追記を表示する
```
sudo tail -f /var/log/filebeat/filebeat.log
```

filebeat.yml(filebeatの設定ファイル)
```
logging.level: debug
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat.log
  keepfiles: 7
  permissions: 0644
```
* logging.level...ログデータの取得範囲。DEBUG>INFO>WARNING>ERRORの順に広い範囲で取得する。
* logging.to_files...ログデータを指定したディレクトリに出力するか。(追記するか)
* logging.files...作成するログファイルのディレクトリや名前を指定。

[elasticsearch/kibana立ち上げ資料](https://github.com/ksaplabo-org/aircondition)

#### docker-composeによる環境構築
1. elasticsearch,kibana,filebeatのイメージを使用してdocker-compose.ymlを作成

  * depends_onの設定をkibana,filebeatに付与。(docker-compsoe内の起動順序)
  * マウント先のパスを設定(./docker-compose_elastic/にそれぞれ作成するよう設計した)
  * filebeatが起動しない

調査事項
* filebeatが起動しない原因(エラーコード、logが確認できない)
  * エラーコード127...コンテナ内のコマンドが見つからない状態  
docker inspect でコンテナの状態を確認  

docker inspectでCmdとEntrypointの項目を確認(run実行時に走るファイルが記載されている)  
runで立ち上げたコンテナに対して上記で確認したファイルが存在するか確認する。有れば走る。  
今回は存在していたので、別の個所に理由がありそう。  

* docker-compose.ymlで作成したイメージとbuildしたイメージの違いを検証。  
→buildしても同様に起動しないことが確認できたので、docker-composeで設定した項目に問題があるかもしれない。

* docker-compose.ymlの記述内容を一つずつ消していく。一つずつ試して、エラーになる原因の項目を探す。  
→原因がvolumeにあることが判明。そもそも、filebeatにymlファイルを更新するためのvolumeマウントが必要ないことが判明。(更新はDockerfileで行う)

* yml更新用のvolume項目を消してイメージを作成。無事に起動したので、次にfilebeatから投入したデータが拾えるかどうかの確認。

* ログをsb-dataに投入しても音沙汰がない。ので、filebeat.yml(設定資料)の確認。  
→そもそも拾ったログをどこに出力するかの設定を記載していなかったため、書く必要がある。
```
filebeat.inputs:
- type: log 
  paths:
    - /var/log/search_log/*.log

output.elasticsearch:
  hosts: '${ELASTICSEARCH_HOSTS:elasticsearch:9200}'
  username: '${ELASTICSEARCH_USERNAME:}'
  password: '${ELASTICSEARCH_PASSWORD:}'

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat.log
  keepfiles: 7
  permissions: 0644
```
上記のymlを書くことで **/var/log/search_log/*.log** にあるログデータが読み取られ、**/var/log/filebeat** 内に **filebeat.log**が作成される。

* docker-compose.ymlを更新して再度イメージから立て直す。

* まだ、ファイルは作成されなかった。理由はfilebeatコンテナにログインしているユーザーが「filebeat」に対して、ファイルの所有者が「root」のため、
アクセス権が「drwxr-xr-x」であるフォルダ・ファイルに書き込むことができない。  
→権限を変更(Dockerfileで以下を記載)
```
USER root
RUN chmod -R 777 /var
USER filebeat
```
アクセス権を変更するためにrootユーザーでログインし、/var以下を「rwxrwxrwx」に変更。再度ユーザーをfilebeatに戻して終了。

* Dockerfileを変更したので、再度イメージから作成。

* まだとりこめなかった。ログデータを確認すると、オフセットが0のままでログデータを読み込むことができていないことが判明。  
aaa.log
```
aaaaa
```
オフセットが0のままなのでエラーを起こしていた。改行を最後にいれてあげることで、エラーは解消された。

* tailコマンドとkibanaの画面でデータが取り込まれていることが確認でき、無事完成。

#### 補足
* 取り込まれるデータが多い!などあった場合や出力先のディレクトリを変えてほしいといった要望があった場合にはfilebeatのymlを更新し、再度イメージから作り直してあげるだけで、問題を解消できる。  
→ただし、filebeatとコンテナの特性上、一度取得したデータかどうか判断するためのデータを永続化しないと、一度elasticsearchに取り込んだでーたを再度拾ってしまう点に注意。

#### 試したこと(無駄足)
→entrypointの上書き(entrypoint: /usr/local/bin/docker-entrypoint)→変数未定義エラー
```
/usr/local/bin/docker-entrypoint: line 7: $1: unbound variable
```

対象コンテナにtty: trueを追加する(疑似ターミナル (pseudo-TTY) を割り当て???)→変わらず。

commitコマンドでコンテナからイメージを作成

#### 不明点
* elasticsearchのulimitsと環境変数(node)...未調査
* elasticsearchのパスにデータが存在しない→パスの間違い/usr/share/elasticsearch/data
* filebeatが立ち上がらないので、ymlを確認できない→イメージをカスタムする必要がある
  * Dockerfileの作成が必要＞

