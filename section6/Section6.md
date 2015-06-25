# Section 6 AWS(Amazon Web Services)

このセクションではAWS(Amazon Web Services)を使用したサーバー構築を行ないます。

## 講義関連リンク

* [AWS公式サイト](http://aws.amazon.com/jp/)
* [Cloud Design Pattern](http://aws.clouddesignpattern.org/index.php/%E3%83%A1%E3%82%A4%E3%83%B3%E3%83%9A%E3%83%BC%E3%82%B8)

## 6-0 AWSコマンドラインインターフェイスのインストール
-----

[公式サイト](http://aws.amazon.com/jp/cli/)参照。

     sudo apt-get install python-pip
     sudo pip --proxy=http://172.16.40.1:8888 install awscli
    
インスタンス作成
セキュリティのところでなんかいろいろやったと思われ。

インスタンス作ったらpemファイルがダウンロードされると思うから、pemファイルダウンロードされたら、権限を400に変える。

下記のコマンドで先生から貰ったアクセスキーもろもろを入力して、コマンドラインから操作できるようにする。

     aws configure



## 6-1	AWS EC2 + Ansible
-----

3-1で作ったAnsibleのもろもろをpemファイルがあるところにコピーしたほうがいいかも。

####3-1で作ったplaybookをAWSで動くように書き換える。

書き換え箇所

- tasks:の上に「user: ec2-user」を追記
- インストールするmysqlの中の「- MySQL-python」を「- MySQL-python27」に書き換える
- 最後らへんのパスに「vagrant」って書かれているところを消す。(2箇所ぐらい)

####hostsファイルのアドレスをEC2のアドレスに書き換える。

####playbookを適用させる

     ansible-playbook -i hosts --private-key ./[pemファイル名] playbook.yml

EC2のアドレスにブラウザからアクセスして確認。wordpressのインストール画面が出てればおｋ。  



: Amazon Elastic Computing Cloud(EC2)を使用してWordpressが動作するサーバーを作ります。
: 3-1を終了している場合、Ansibleで構築できるようになっているのでAnsibleを使って構築する。
: 終わってない場合は手動でがんばってね。

### AMI(Amazon Machine Image)を作る

自分のインスタンスを選択した状態で、アクションからイメージの作成を選択(インスタンス右クリックでも出てくる)。

イメージ名はとりあえず「学籍番号_AMI」にした。

インスタンスを再起動後、wordpressが表示されればおkだったはず。


環境の構築が終わったら、AMIを作成します。AMIを作成後、同じマシンを2つ起動して、コピーができていることを確認してください。

## 6-2 AWS EC2(AMIMOTO)
-----

インスタンスの作成からAWS Marketplaceを選択、amimotoで検索してamimoto(HHVM)を選択。

で、そのままインスタンス作成して、接続確認。



6-1では自力(?)で環境構築を行ない、AMIを作成したが、別の人が作ったAMIを使用してサーバーを起動することもできる。

AMIMOTOのWordpressを起動してWordpressが見れることを確認する。

## 6-3 Route53
-----

Route53に行ってHosted ZonesでCreate Hosted Zoneし、Import Zone Fileに5-1でやったzone fileをコピペしてCreateして終わり!!



Route53はAWSが提供するDNSサービス。

5-1で作ったDNSの情報をRoute53に突っ込んでみよう。

## 6-4 S3
-----
コマンドラインから  
バケッドを作る

     aws s3 md se://n14004

これでn14004というバケッドが出来てる。
適当に作ったWebサイトをアップロード

     aws s3 cp --acl public-read index.html s3://n14004/index.html
     
     ※--acl public-read を書かないとアップロードはされるけどWebサイトが見れない。



S3はSimple Storage Service。その名の通り、ファイルを保存し、(状況によっては)公開するサービス。

てきとーにWebサイトを作り、それをS3にアップロードし、公開してみよう。

S3にアップロードする際にはAWSコマンドラインインターフェイスを使ってね。

## 6-5 CloudFront
-----

6-1で作ったAMIを起動。

CloudFrontの「Create Distribution」からWebで登録。

「Origin Domain Name」のところにEC2の起動したAMIのパブリックDNSをコピペ。

「Origin ID」は適当に何か書く。(今回は学籍番号にした)

下にスクロールすると「Forward Query Strings」って項目があるから、それをYesにする。

あとはCreate Distributionを押すだけ。完全に起動するまで時間かかる。



CloudFrontはCDNサービスです。CDNって何って?ggrましょう(ちゃんと講義では説明するので聞いてね)。
6-1で作ったAMIを起動し、CloudFrontに登録します。登録して直接アクセスするのとCloudFront経由するのどっちが速いかベンチマークを取ってみましょう。

また、CloudFrondを経由することで、地域ごとにアクセス可能にしたり不可にしたりできるので、それを試してみましょう。

## 6-6 RDS
-----

AWSマネジメントコンソールからRDS選択。DBインスタンスの起動。

MySQLを選択。

いいえを選択。

DBインスタンスのクラスを一番最初のmicroって書かれているやつに変更。(安いらしい)

マルチAZ配置→はい

ストレージタイプをマグネティックに。

パブリック・アクセス可能をいいえに変更。

データベースの名前を入れてインスタンス作成。

EC2のインスタンスにsshで接続してRDSのmysqlにログインしてなにも触らずログアウト。

/usr/share/nginx/wordpress/wp-config.phpを削除してブラウザでWordPressを起動、データベースをlocalhostからRDSのDB名に変更

ブラウザからアクセスして動いたらおｋ。



RDSは…MySQLっぽい奴です。

RDSを立ち上げて、6-1で作ったAMIのWordpressのDBをRDSに向けてみよう。

## 6-7 ELB
-----

ELBはロードバランサーです。すごいよ。

6-1で作ったAMIを3台ぶんくらい立ち上げてELBに登録し、負荷が割り振られているか確認してみよう。

## 6-8 API叩いてみよう
-----

AWSは自分で作ったプログラムからもいろいろ制御できます!
なんでもいいのでがんばってプログラム書いてみてね(おすすめはSES)。
