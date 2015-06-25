# Section 3 Ansibleによる自動化とテスト
-----

毎回毎回手動で

    yum install もにょもにょ

とやるのも非効率なので、それらを自動化してくれるツールを使って今迄の作業を何回でもできるようにします。

今回の講義ではAnsibleを使用します。

## 3-0 Ansibleのインストール

[公式サイト](http://docs.ansible.com/intro_installation.html#latest-releases-via-apt-ubuntu)に手順載ってる。

     sudo apt-get install software-properties-common
     sudo apt-add-repository ppa:ansible/ansible
     sudo apt-get update
     sudo apt-get install ansible

ホスト側(Ubuntu)にインストール。

## 3-1 ansibleでWordpressを動かす(2)を行なう
-----

hostsファイルを作る

     echo 192.168.56.148 > hosts

確認でping  
pingのとき打ったコマンド

     ansible 192.168.56.141 -i hosts -m ping --private-key ~/.vagrant.d/insecure_private_key -u vagrant -k
     
sshpassインストールしろーとか言われたらインストールする。
passwordデフォルトは「vagrant」

※上記は確認なのでさほど必要な行為ではない。

Ansible用に書いたVagarantファイルと  
書いたhostsファイルとかplaybookとか別のところに置いとくので、そこ参照してください…。

下記コマンドでplaybookを適用。

     ansible-playbook playbook.yml -i hosts  --private-key ~/.vagrant.d/insecure_private_key -u vagrant -k

ブラウザからアクセスして、wordpressインストール画面が出たらおｋ。


2-1でWordpressをNginx + PHP + MariaDBでインストールした手順をAnsibleのPlaybookで実行するように記述し、動かしてみてください。
