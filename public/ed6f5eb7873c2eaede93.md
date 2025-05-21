---
title: Docker で MySQL コンテナーがなぜか起動できない問題
tags:
  - MySQL
  - Ubuntu
  - Docker
private: false
updated_at: '2017-06-27T19:24:13+09:00'
id: ed6f5eb7873c2eaede93
organization_url_name: null
slide: false
ignorePublish: false
---
Ubuntu 上でなぜか MySQL コンテナーが起動せず、ハマりました。ワークアラウンドをメモしておきます。


再現環境
----
- Ubuntu 16.04.2 LTS 64bit (on Vagrant)


現象
----
以下のような docker-compose.yml を作成して、 `docker-compose up -d db` で起動しようとしても、 `Exit 1` で終了してしまいます。

```yaml:docker-compose.yml
version: '2'
services:
db:
    image: mysql:5.6
    ports:
        - "3302:3306"
    environment:
        - TZ=JST-9
        - MYSQL_USER=foo
        - MYSQL_PASSWORD=foo
        - MYSQL_ROOT_PASSWORD=foo
        - MYSQL_DATABASE=foo
    volumes:
        - ~/foobar/db/mysql/dump/:/docker-entrypoint-initdb.d/
        - ~/foobar/db/mysql/conf/:/etc/mysql/conf.d
    privileged: true
    networks:
        - datastore
```

起動してみる:

```sh
$ docker-compose up -d db
Starting db

$ docker-compose ps
     Name                  Command             State            Ports          
------------------------------------------------------------------------------
db               docker-entrypoint.sh mysqld   Exit 1 
```

起動時のエラーログは以下:

```sh
$ docker-compose logs db
Attaching to db
db  | 
db  | ERROR: mysqld failed while attempting to check config
db  | command was: "mysqld --verbose --help --log-bin-index=/tmp/tmp.BTfnm2u97W"
db  | 
db  | mysqld: error while loading shared libraries: libpthread.so.0: cannot open shared object file: Permission denied
db  | 
db  | ERROR: mysqld failed while attempting to check config
db  | command was: "mysqld --verbose --help --log-bin-index=/tmp/tmp.Q0noJONZOp"
db  | 
db  | mysqld: error while loading shared libraries: libpthread.so.0: cannot open shared object file: Permission denied
```

なぜか共有ライブラリが `Permission Denied` でエラーとなっています。


原因
----
Ubuntu (Debian) では AppArmor というセキュリティ機構が動いており、これが Docker の priviledged モードを阻害していました。

`privileged: true` にすると、 Docker 内のコンテナーがホストとほぼ同じ権限を持つことができ、例えば Docker in Docker のような環境を構築できます。一方でコンテナー外のリソースまでアクセス可能になってしまい、コンテナー間の疎結合を壊してしまうので、開発用途などを除いて通常は使いません。

ホスト OS 上に MySQL がインストールされている状態で、 Privileged モードを有効にして MySQL コンテナを起動すると、 AppArmor が「許可していないリソース上で MySQL が動作している」と認識し、実行をブロックするようです。


解消方法
----
以下のいずれかの方法で解消が可能です。

### ホストから MySQL を削除する

```sh
sudo apt remove mysql-server
```

### AppArmor の監視から MySQL を除外する
```sh
sudo ln -s /etc/apparmor.d/usr.sbin.mysqld /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.mysqld
# (単純に /etc/apparmor.d/usr.sbin.mysqld を rm しても OK ですが、上記のようにするのが Ubuntu の流儀)
```

複雑な問題ですね。


補足
----
そもそも Privileged を有効にする必要性がよくわかりませんでした。 MySQL コンテナーならデータを入れるだけだし、 networking さえ設定しておけば Privileged は無効にしたままで十分だと思うんですが…。

別チームの人がこの docker-compose.yml を書いたんですが、何がしたかったんでしょうか？今度問い詰めてみます。


参考
----
- [Mysql, Privileged mode, cannot open shared object file · Issue #7512 · moby/moby](https://github.com/moby/moby/issues/7512)
  - https://github.com/moby/moby/issues/7512#issuecomment-61787845 がワークアラウンド
- https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities
- [Docker privileged オプションについて - Qiita](http://qiita.com/muddydixon/items/d2982ab0846002bf3ea8)
- [AppArmor security profiles for Docker | Docker Documentation](https://docs.docker.com/engine/security/apparmor/)
