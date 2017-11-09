# 爆速WEBサーバ"H2O"に触れてみよう
2017.11.9 サポーターズ勉強会資料

# 環境手順

* 任意の作業フォルダを作成して移動する

`mkdir h2o ; cd h2o`

* Ansibleのインストール

`sudo pip install ansible`

* gitのインストール

`sudo yum install git`

* リポジトリを任意をフォルダにクローンする。

`git clone https://github.com/VTRyo/handson-amazonlinux.git`

* ディレクトリを移動する

`cd handson-amazonlinux/ansible`

* ansible実行

`/usr/local/bin/ansible-playbook -i hosts playbook.yml`

## 他うまくいかない場合は手動(CentOS系)

* yumレポジトリを登録する

`vim /etc/yum.repos.d/bintray-tatsushid-h2o-rpm.repo`

```
#bintray-tatsushid-h2o-rpm - packages by tatsushid from Bintray
[bintray-tatsushid-h2o-rpm]
name=bintray-tatsushid-h2o-rpm
#If your system is CentOS
#baseurl=https://dl.bintray.com/tatsushid/h2o-rpm/centos/$releasever/$basearch/
#If your system is Amazon Linux
baseurl=https://dl.bintray.com/tatsushid/h2o-rpm/centos/6/$basearch/
gpgcheck=0
repo_gpgcheck=0
enabled=1
```

* インストール実行

`sudo yum install h2o`

* サービス起動

`/etc/init.d/h2o start`

* phpインストール

`sudo rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm`

`sudo rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm`

`sudo yum -y install libwebp --disablerepo=amzn-main --enablerepo=epel`

`sudo yum -y install httpd libmcrypt libtool-ltdl libtidy libXpm libtiff autoconf automake gd-last enchant-aspell`

`sudo yum -y install php php-fpm --disablerepo=amzn-main --enablerepo=remi-php70`

* サービス起動

`/etc/init.d/php-fpm start`

# 共通手順

### ブラウザでローカルホストにアクセスしてみる

* 任意のブラウザ（Chrome、Firefox、など）

`<IPadress>`

### php-fpmと連携してみる

* /etc/php-fpm.d/www.confの編集

・追記
listen = /var/run/php-fpm/php-fpm.sock

・変更
listen.owner = nobody ;コメントアウトはずす
listen.group = nobody ;コメントアウトはずす

```
; Start a new pool named 'www'.
[www]

; The address on which to accept FastCGI requests.
; Valid syntaxes are:
;   'ip.add.re.ss:port'    - to listen on a TCP socket to a specific address on
;                            a specific port;
;   'port'                 - to listen on a TCP socket to all addresses on a
;                            specific port;
;   '/path/to/unix/socket' - to listen on a unix socket.
; Note: This value is mandatory.
;listen = 127.0.0.1:9000 コメントアウトする
listen = /var/run/php-fpm/php-fpm.sock

;~~~~省略

; Set permissions for unix socket, if one is used. In Linux, read/write
; permissions must be set in order to allow connections from a web server. Many
; BSD-derived systems allow connections regardless of permissions.
; Default Values: user and group are set as the running user
;                 mode is set to 0666
listen.owner = nobody ;コメントアウトはずす
listen.group = nobody ;コメントアウトはずす
;listen.mode = 0666
```

* /etc/h2o/h2o.confに追記する

```
file.custom-handler:
  extension: .php
  fastcgi.connect:
    port: /var/run/php-fpm/php7.0-fpm.sock
    type: unix
```

* サービス再起動

`/etc/init.d/php-fpm restart`

`/etc/init.d/h2o restart`

* 通信確認

`netstat -a --unix | grep php-fpm`

### basic認証設定してみる

* /etc/h2o/h2o.confを以下のように変更する

```
user: nobody

file.custom-handler:
  extension: .php
  fastcgi.connect:
    port: /var/run/php-fpm/php7.0-fpm.sock
    type: unix

hosts:
  "localhost:443":
    listen:
      port: 443
      host: 0.0.0.0
      ssl:
        certificate-file: "/etc/pki/tls/certs/localhost.crt"
        key-file: "/etc/pki/tls/private/localhost.key"
    paths:
      "/":
        file.dir: /var/www/html
  "localhost:80":
    listen:
      port: 80
      host: 0.0.0.0
    paths:
      "/":
        mruby.handler: |
         require "htpasswd.rb"
           acl {
             allow { addr == "0.0.0.1" }
             deny { user_agent.match(/curl/i) && ! addr.start_with?("192.168.") }
             use Htpasswd.new("/var/www/html/.htpasswd", "realm-name") { path.start_with?("/") }
           }

        file.dir: /var/www/html
access-log: /var/log/h2o/access.log
error-log: /var/log/h2o/error.log
pid-file: /var/run/h2o/h2o.pid
```

* syntaxチェック

`h2o -t -c /etc/h2o/h2o.conf`

* サービス再起動

`/etc/init.d/h2o restart`

* 任意のブラウザにて認証が表示されるか見る（Chrome、Firefox、など）

`<IPadress>`
