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

* discovery.type: single-nodeの追記...**デフォルトでmulti-node。複数ノードのクラスターを作成するか、単一ノードのクラスターを作成するか。**

* systemd...コンピュータのシステムを起動するときに様々なプログラムを動かす、元のプログラムのこと。  
→systemdに追加することで起動時にプログラムを動かしてくれる。

* **daemon-reload**...enableとセットで使用されているように見える。どのような命令か不明。

* rpm...linuxのパッケージ管理コマンド。yumと似ている。**違いは?**
  * rpm...パッケージ個々の管理をするコマンド。依存関係を考慮しない。パッケージ個々情報表示がyumより詳細である。
  * yum...パッケージの依存関係を考慮してインストールしてくれる。**基本yumでよい?rpmを使用するタイミングは?**

  * rpm...オフラインでのインストール。curlでGETしたものをオフラインでインストールしている。(YUMの過程にある鍵のインポートのみ例外)
  * yum...オンラインでのインストール(repoファイルに記載したURLからインストール)

YUMの手順
1. 公開鍵を取得。
2. URLを辿ってインストール。
3. 公開鍵を使用してハッシュ値を比較。正しければ

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

elasticsearch/kibana立ち上げ資料  
https://github.com/ksaplabo-org/aircondition
