# Section 2 その他のWebサーバー環境

## 2-1 Vagrantを使用したCentOS6.5環境の起動
-----

### Vagrantで起動できるCentOS 6.5のイメージを登録

先生のUSBストレージからVagrant用CentOS boxをコピーし、それを登録する。

     vagrant box add CentOS65 コピーしたboxファイル --force
    
###Vagrantの初期設定

作業用ディレクトリを作成し、その中で初期設定を行う。

     vagrant init
    
上記コマンドを実行すると、Vagrantfileというファイルが作成される。このファイルを書き換え、先ほど登録したCentOS6.5を起動するようにする。

viでVagrantfileを開き、

     config.vim.box = "base"
     ↓これを
     config.vim.box = "CentOS65"
    
に書き換える。ここで指定するのはvagrant box addで指定したものの名前(この場合CentOS65)になる。

###Vagrantのコマンド

仮想マシンの起動

     vagrant up

仮想マシンの停止

     vagrant halt
     
仮想マシンの一時停止

     vagrant suspend
     
仮想マシンの破棄

     vagrant destroy
     ※いろいろ設定したけど最初からやり直したい…そんな時に破棄するとCentOSが初期化される。またvagrant upすると立ち上がる。
     
仮想マシンへ接続

     vagrant ssh
     ※実際の仮想マシンへはsshで接続する。
     
###ホストオンリーアダプターの設定

サーバーを設定したあと、動作確認するために接続するためのIPアドレスを設定。またそのためのNICを追加。

Vagrantfileの

    Vagrant.configure(2) do |config|

から一番最後の

    end

の間に

    config.vm.network "private_network", ip:"192.168.56.129"

と書くと仮想マシンのNIC2に192.168.56.129のIPアドレスが振られる。
`config.vm.box = "CentOS7"` の下にでも書くといいと思います。

※ 当然のことながら、複数台の仮想マシンを立ち上げる時には異なるIPアドレスを割り当てる必要がある。

### Vagrantfileの反映

Vagrantfileで変更した設定を反映させるには

    vagrant reload

すると反映される。ただし、再起動されるから注意。



## 2-2 Wordpressを動かす(2)
-----

1-2ではWordpressをApache + PHP + MySQLで動作させたが、今度はNginx + PHP + MariaDBで動作させる。    
OSはCentOS6.5。CentOS7じゃないよ。

###Nginxをインストール

     "Nginxはディストリビューターからrpmが提供されていないため、リポジトリを追加する必要があります。
     [公式サイト](http://nginx.org/en/linux_packages.html#stable)からリポジトリ追加用のrpmをダウンロードしてインストールしてください。
     その後yumでインストールできるようになります。"
     
ってかかれていたけど、どうしてもリポジトリ追加用のrpmをダウンロードできなかったので、直接書いた。

nginx.repoというファイルを/etc/yum.repos.d/配下につくる。

     vi nginx.repo
     
で下記を書く。

     [nginx]
     name=nginx repo
     baseurl=http://nginx.org/packages/centos/6/$basearch/
     gpgcheck=0
     enabled=1
     
保存したら、その後yumインストールできた。

     yum -y install nginx
     
###MariaDBをインストール

これもリポジトリが必要なのでレポジトリを設定する。

     vi /etc/yum.repos.d/mariadb.repo
     ↓以下ファイルの中身
     [mariadb]
     name = MariaDB
     baseurl = http://yum.mariadb.org/5.5/centos6-amd64
     gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
     gpgcheck=1

インストールする前に一度yum updateしたほうがいいはず…。    
yum updateする前に、vi /etc/yum/pluginconf.d/fastestmirror.confを開き｢prefer=ftp.riken.jp｣を追記してupdateをできるだけ早くしとく。    
yum search MariaDBとかしてあるかどうか確認してみるとか。

設定したらインストール開始

     yum install MariaDB-client MariaDB-server
     
###MariaDB設定

やってることはmySQLと変わらない。

mysql -u rootでmysql開く。

    mysql> create database wpdb; (wordpress用DBを作る。この場合"wpdb"というDB作った。)
    mysql> grant all privileges on wpdb.* to n14004@localhost identified by 'password';　(作ったDBに接続できるユーザー名とパスワードを設定。)
    
    ※SELECT host,user FROM mysql.user;
    で作成したユーザーを確認できる。
     
###PHPをインストール

     sudo yum -y install php-mysql php php-gd php-mbstring php-fpm
     
###php-fpmを設定、起動

/etc/php-fpm.d/www.confの中を編集。

     user = nginx (デフォルトだとapacheって書いてるからnginxに書き換える)
     group = nginx (上と同じく)
     
起動

     /etc/init.d/php-fpm start
     
OS起動時に自動起動するようにする。

     chkconfig php-fpm on
     
###/etc/nginx/conf.d/default.confを編集

    location / {
        root   /usr/share/nginx/html;
        index  index.php; 　(index.htmlからindex.phpに変える。追加でも可)
    }
    
下記をどこかに追加

    try_files $uri $uri/ /index.php?q=$uri&$args;

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
        root           /usr/share/nginx/html;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }

###Nginxを起動

起動

     sudo service nginx start
     
OS起動時に自動起動するようにする。

     chkconfig nginx on
     
Webコンテンツを置くドキュメントルートは/usr/share/nginx/html/になる。

起動したらブラウザからhttp://192.168.56.129にアクセスしてみて確認。たぶんnginxなんたらって出るはず。

###wordpressをダウンロード、インストール

wgrepをyum install。そんで/etc/wgetrcにいってプロキシ設定したらwordpressをwgetする。

     wget http://ja.wordpress.org/wordpress-4.2.2-ja.tar.gz
     tar zxvf wordpress-4.2.2-ja.tar.gz
     
展開したwordpressを/usr/share/nginx/htmlに移動。
wordpress内のwp-config-sample.phpをwp-config.phpに名前を変えてコピーし、sqlで設定したデータベース名・ユーザー名・パスワードを入力。

ブラウザからアクセス。
http://192.168.56.101/wordpress/wp-admin/install.php

あとはインストールしたらおーけー。


## 2-3 Wordpressを動かす(3)
-----

ディレクトリを新しく作り、そこでvagrant initして新しく作る。行程は2-1と同じ。    
(※IPアドレスも同じなのでどちら開くときは必ず片方をvagrant haltして使う。それかIPアドレス変えるか。)

起動したら、まずyumのプロキシを設定する。    
wgetをyumでインストール、wgetもプロキシ設定する。    
vi /etc/yum/pluginconf.d/fastestmirror.confを開き｢prefer=ftp.riken.jp｣を追記し、yum updateする。    

###Apache HTTP Server 2.2

下記でapacheのダウンロード、展開。

     wget http://ftp.meisei-u.ac.jp/mirror/apache/dist//httpd/httpd-2.2.29.tar.gz
     tar zxvf httpd-2.2.29.tar.gz

展開したディレクトリに入り、

     ./configure
     
を実行。その後

     make
     sudo make install
     
でインストール。

###apache設定

vi /usr/local/apache2/conf/httpd.confで開き  
<IfModule dir_module>のところに｢index.php」を追記する  

次に以下の記述を追加する
  
     <FilesMatch "\.ph(p[2-6]?|tml)$">  
        SetHandler application/x-httpd-php  
     </FilesMatch>  

     <FilesMatch "\.phps$">  
        SetHandler application/x-httpd-php-source  
     </FilesMatch>`

起動する

     /usr/local/apache2/bin/apachectl start

###php

wgetでダウンロード、展開。

     wget http://php.net/distributions/php-5.5.25.tar.gz
     tar zxvf php-5.5.25.tar.gz

展開したディレクトリに移動。
ここでインストール始めようとすると「libxml2」がないと言われるので、ここで入れとく。

     yum install libxml2-devel
     
多分これでlibxml2は入ったと思うので、phpインストールする。

     ./configure --with-apxs2=/usr/local/apache2/bin/apxs --with-mysql
     make
     sudo make install
     
上記の順で完了。

###php設定

ファイルをコピーする。

     cp php-5.5.25/php.ini-development /usr/local/lib/php.ini
     
コピーした場所をviで開きファイルを編集。
     
     ; http://php.net/pdo_mysql.default-socket
     pdo_mysql.default_socket="/var/lib/mysql/mysql.sock"
     
     ; http://php.net/mysql.default-socket
     mysql.default_socket="/var/lib/mysql/mysql.sock"
    
     ※""でくくってるところを追記

###mysql

     yum install mysql

###mysql設定

mysql -u rootでmysqlスタート。    
データベース、ユーザー名、パスワードを設定(2-2とかでやった時と一緒)。

起動する

     service mysqld start

###wordpress

2-2のwordpressの動作の手順と同じ。
wordprassの場所は/usr/local/apache2/htdocsに移動させる。

apacheとmysqlを再起動

     service mysqld restart
     /usr/local/apache2/bin/apachectl restart

wordpressにアクセスする。

## 2-4 ベンチマークを取る
-----

ホストにabコマンドをインストール。

     sudo apt-get install sudo apache2-utils
     
abコマンドで負荷テストする。

###wordpressにプラグイン追加

2-2で立ち上げたマシンで試す。     
wordpressの管理画面行き、プラグイン「WP Super Cache」を追加。
入れたらプラグインを有効化する。
下記のコマンドで負荷テスト。

     ab -n 10 -c 10 http://192.168.56.129/wordpress/
     
     
プラグイン有効化する前

     Server Software:        nginx/1.8.0
     Server Hostname:        192.168.56.129
     Server Port:            80

     Document Path:          /wordpress/
     Document Length:        9168 bytes

     Concurrency Level:      10
     Time taken for tests:   2.589 seconds
     Complete requests:      10
     Failed requests:        0
     Total transferred:      93850 bytes
     HTML transferred:       91680 bytes
     Requests per second:    3.86 [#/sec] (mean)
     Time per request:       2589.344 [ms] (mean)
     Time per request:       258.934 [ms] (mean, across all concurrent requests)
     Transfer rate:          35.40 [Kbytes/sec] received

     Connection Times (ms)
                   min  mean[+/-sd] median   max
     Connect:        1    1   0.1      1       1
     Processing:  1882 2165 252.8   2112    2588
     Waiting:     1688 2062 309.5   2002    2582
     Total:       1883 2166 252.9   2113    2589

     Percentage of the requests served within a certain time (ms)
       50%   2113
       66%   2119
       75%   2320
       80%   2583
       90%   2589
       95%   2589
       98%   2589
       99%   2589
      100%   2589 (longest request)

-----

プラグイン有効化した後。

     Server Software:        nginx/1.8.0
     Server Hostname:        192.168.56.129
     Server Port:            80

     Document Path:          /wordpress/
     Document Length:        9168 bytes

     Concurrency Level:      10
     Time taken for tests:   9.617 seconds
     Complete requests:      10
     Failed requests:        0
     Total transferred:      93850 bytes
     HTML transferred:       91680 bytes
     Requests per second:    1.04 [#/sec] (mean)
     Time per request:       9616.878 [ms] (mean)
     Time per request:       961.688 [ms] (mean, across all concurrent requests)
     Transfer rate:          9.53 [Kbytes/sec] received

     Connection Times (ms)
                   min  mean[+/-sd] median   max
     Connect:        1    1   0.0      1       1
     Processing:  8389 8867 506.8   8655    9615
     Waiting:     8177 8467 360.4   8287    9080
     Total:       8391 8869 506.8   8656    9617

     Percentage of the requests served within a certain time (ms)
       50%   8656
       66%   9123
       75%   9489
       80%   9504
       90%   9617
       95%   9617
       98%   9617
       99%   9617
      100%   9617 (longest request)



サーバーの性能測定のためにベンチマークを取ることがあります。

## 2-5 セキュリティチェック
-----

サーバーを構築したとしても、セキュリティがガバガバではいろんな意味で駄目です。
Webアプリケーションの脆弱性を突かれたり、設定したサーバーに脆弱性があったりした場合、
情報漏洩とか乗っ取りとか踏み台とかされるとアレです。

定期的にセキュリティチェックを行なう必要があります。

(やり方は後で)

セキュリティチェックを行ない、不具合があるようでしたら修正を行なって再度セキュリティチェックを行ないます。
(可能な限り不具合がなくなるまでチェック&fixを行ないます。)
