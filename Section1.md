
# Section 1 基本のサーバー構築

## 1-1 CentOS 7のインストール
-----

### VirtualBoxへのインストール

1. [CentOSの公式サイトのダウンロードページ](https://www.centos.org/download/)よりCentOS 7 Minimal ISO(x86_64)のISOファイルをダウンロードする。    
(Actual Countryのhttp://ftp.riken.jp/Linux/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1503-01.iso を選択)

2. VirtualBox上にインストールする。
    * 新規を作成する。(名前:CentOS7　タイプ:linux　バージョン:Red Hat(64bit))
    * メモリサイズ1024MB,ストレージの容量は8.00GB
    * ファイルの環境設定を開き、ネットワーク→ホストオンリーネットワーク→ホストオンリーネットワークを追加。
    * 設定を開く。
    * ストレージ→コントローラー:IDEの空を選択。属性のCD/DVDのアイコンをクリックして前でダウンロードしたiosファイル選択。
    * ネットワークでネットワークアダプター2を設定。ネットワークアダプタ1はデフォルトで大丈夫。
    * 起動し、インストール開始。
    * インストール中に指示されるパーティションの設定は特に指定なし。
    * インストール中、root以外の作業用(管理者のユーザを作成)
    * インストール完了


### ネットワークアダプター1/2へのIPアドレスの設定とssh接続の確認

/etc/sysconfig/network-scriptにifcfg-enp0s3とifcfg-enp0s8というファイルがあるので、
そのファイルを編集してネットワーク接続ができるように設定。    
それぞれファイルの中身を下記の内容に設定。(DHCPでIPアドレス取得する)

    DEVICE=(ファイルがifcfg-enp0s3なら"ifcfg-enp0s3",ifcfg-enp0s8なら"ifcfg-enp0s8")    
    BOOTPROTO=dhcp
    ONBOOT=yes

[RedHat Enterprise Linux 7のマニュアル(英語)](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html-single/Networking_Guide/index.html#sec-Configuring_a_Network_Interface_Using_ifcg_Files)に詳しいこと書いてる。

設定したら、

    $ ip addr
してIPアドレスを確認。

### SSH接続の確認

Ubuntuからsshで仮想マシンに接続できることを確認。

    $ ssh 192.168.56.101
で接続できる。(IPアドレスは上記で確認したIPアドレスで)

(ついでなので、公開鍵認証でログインできるようにしておくといいと思いますよ。必須ではないけど。)

### インストール後の設定

yumやwgetを使用する時のproxyの設定。

1. yumの設定    
    $ sudo vi /etc/yum.conf
     
    以下を追記  
    proxy=http://172.16.40.1:8888/    
    proxy=https://172.16.40.1:8888/ 
    
2. wgetの設定    
    $ sudo vi /etc/wgetrc 
    
    ＃You can set the default proxies for Wget to use for http and ftp.    
    ＃They will override the value in the environment.    
    http_proxy = http://172.16.40.1:8888/    
    https_proxy = http://172.16.40.1:8888/    
    
    常にプロキシを使用するか否かの設定    
    ＃If you do not want to use proxy at all, set this to off.    
    use_proxy = on
    
変更したら再起動する。
    
[参考サイト](http://ry.tl/proxy_user.html)

### アップデート

アップデートする。

    $ yum update

## 1-2 Wordpressを動かす(1)
-----

ssh接続できるようになっているので、VirtualBoxの画面からではなく、UbuntuからSSHで接続して設定する。
(そのほうが圧倒的に楽。)

###下記のソフトウェアをインストール。 [※1](#LAMP)

* Apache HTTP Server
    - $ sudo yum install httpd 
    
* MySQL
    - CentOS7にデフォルトで入っているMariaDBをアンインストール    
    $ sudo yum -y remove mariadb    
    $ sudo yum -y remove mariadb-libs    
※無ければこの作業は飛ばしても可

    - MySQL公式リポジトリファイルをインストール    
    $ sudo yum -y install http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm

    - MySQLインストール    
    $ sudo yum -y install mysql mysql-devel mysql-server mysql-utilities

* PHP
    - $ sudo yum -y install php-mysql php php-gd php-mbstring
    
<a name="LAMP">※1</a>: Linux・Apache・MySQL・PHPの頭文字を取ってLAMPという。

###Apache起動

    $ sudo systemctl start httpd.service
    
※OS起動時に一緒に起動させときたいから以下も設定する。

    $ sudo systemctl enable httpd.service
    
###MySQLの起動と設定

    $sudo systemctl start mysqld
    
※OS起動時に一緒に起動させときたいから以下も設定。

    $ sudo systemctl enable mysqld
    
    
    
$ mysql -u rootでmysql開く。

    mysql> create database wpdb; (wordpress用DBを作る。この場合"wpdb"というDB作った。)
    mysql> grant all privileges on wpdb.* to n14004@localhost identified by 'password';　(作ったDBに接続できるユーザー名とパスワードを設定。)
    
    ※SELECT host,user FROM mysql.user;
    で作成したユーザーを確認できる。
    
###Wordpressをダウンロード、展開、設定

    $ wget http://ja.wordpress.org/wordpress-4.2.2-ja.tar.gz
    $ tar zxvf wordpress-4.2.2-ja.tar.gz
    
展開したwordpressを/var/www/htmlに移動。

    $ mv wordpress /var/www/html
    
wordpress内のwp-config-sample.phpをwp-config.phpに名前を変えてコピーする。
wp-config.phpファイルを編集する。以下を変更。

    define('DB_NAME', 'wpdb');　→wordpress用のデータベース名
    define('DB_USER', 'n14004');　→データベースのユーザー名
    define('DB_PASSWORD', 'hoge');　→データベースのパスワード
    define('DB_HOST', 'localhost');
    
###SELINUX変更する

$ sudo vi /etc/selinux/configして、以下に変更。

    SELINUX=disabled
    
###firewalldを止める

$ sudo systemctl stop firewalld

でファイアーウォール止める。

###wordpressのインストール

ブラウザからhttp://192.168.56.101/wordpress/wp-admin/install.phpにアクセスする。(今回はIPアドレスだけどIPアドレスの部分は違かったりする)    
表示すれば、あとは手順どおり設定しインストール完了。    



