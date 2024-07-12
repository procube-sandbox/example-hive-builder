# CITSIDaaS 構築ガイド(productionステージ版)

hive-builder では、 private, staging, production の３段階のステージングが可能ですが、この構築ガイドでは production 環境の構築手順を示します。

idaas.procube-demo.jp については inventory/group_vars/all.yml で domain 変数に設定している値に読み替えてください。

## 1. mother マシン構築

Github リポジトリ https://github.com/procube-sandbox/example-hive-builder をforkして、 CodeSpaces を起動してください。 CodeSpaces の環境が mother マシンとなります。

### 2. hive環境の設定
以下のコマンドで hive の環境を設定してください。

    hive set stage production
    hive install-collection

コマンドの実行時に以下のようなメッセージが表示される場合がありますが、hive コマンドでは、 ansible 起動時にコンテキストディレクトリの collections を collection の検索パスに指定しますので、問題ありません。無視してください。

    [WARNING]: The specified collections path '/root/hive-idaas/.hive/production/collections' is not part of the configured Ansible collections paths '... '. The installed collection won't be picked up in an Ansible run.

### 4. AWS のアカウント
hive-builder のマニュアルの[AWSの準備](https://hive-builder.readthedocs.io/ja/latest/aws.html)の章の手順 1, 2 にしたがって、アクセスキーを取得して hive の環境に設定してください。
t3.large を 4 台起動しますので、AWS見積ツールなどで費用を把握してください。


### 5. 秘密情報ファイル(credentials.yml)の作成
パスワードなどの秘密情報を保持するファイルを \~/.hive/credentials.yml という名前で作成してください。~/.hive というディレクトリがなければ作成してください。credentials.yml の中に以下のような形式で秘密情報を設定してください。

~/.hive/credentials.yml

     dockerhub_login_user: dockerhubのユーザID
     dockerhub_login_password: dockerhubのパスワード
     acme_email: 自分のメールアドレス
     nssdc_client_cert: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
     nssdc_client_key: |
       -----BEGIN RSA PRIVATE KEY-----
       ...
       -----END RSA PRIVATE KEY-----
     ddclient:
       ns0-idaas.procube-demo.jp:
         name: ns0のID
         password: ns0のパスワード
       ns1-idaas.procube-demo.jp:
         name: ns1のID
         password: ns1のパスワード
       ns2-idaas.procube-demo.jp:
         name: ns2のID
         password: ns2のパスワード

上記ファイルの準備方法については、以下のとおりです。

#### 5-1. dockerhub のアカウント
[dockerhub](https://hub.docker.com/) のアカウントの ID/Password を　credentials.yml の中で dockerhub_login_user 変数と dockerhub_login_password 変数に設定してください。
アカウントをお持ちでない場合は、Sign up でアカウントを作成してください。

#### 5-2. NSSDC のクライアント証明書と秘密鍵
ユニリタソフトウェア配布センタにアクセスするためのクライアント証明書と秘密鍵をcredentials.yml の中で nssdc_client_cert 変数と nssdc_client_key 変数に設定してください。
お持ちでない場合は申請してください。証明書と鍵は PEM形式で指定してください。 | のあとに開業して2桁インデントして設定してください。

#### 5-3. サブドメインの委任と ddclient の設定
inventory/group_vars/all.yml の domain 変数にサイトのドメインの fqdn を設定してください。
また、親ドメインに以下のレコードを追加してください。

| ホスト名         | タイプ | TTL   | 値               |
| --------------- | ------| ----- | ---------------- |
| idaas.procube-demo.jp    |   NS  | 1時間  | ns0-idaas.procube-demo.jp |
|                 |       |       | ns1-idaas.procube-demo.jp |
|                 |       |       | ns2-idaas.procube-demo.jp |

##### 5-3(A) 親ドメインをGoogle Domains で管理している場合
親ドメインをGoogle Domains で管理している場合、ダイナミックDNSとして以下を登録してください。

| ホスト名         | タイプ | TTL   |
| --------------- | ------| ----- |
| ns0-idaas.procube-demo.jp |   A  | 1分  |
| ns1-idaas.procube-demo.jp |   A  | 1分  |
| ns2-idaas.procube-demo.jp |   A  | 1分  |

ダイナミックDNSに登録したレコードの認証情報のユーザ名とパスワードを credentials.yml の中の ddclient 変数のホストごとのオブジェクトの name 属性と password 属性に上記例に従って設定してください。

##### 5-3(B) 親ドメインをGoogle Domains 以外で管理している場合
親ドメインをGoogle Domains 以外で管理している場合、 ddclient は使用できません。 credentials.yml の中の ddclient 変数は削除し、構築後に Elastic IP を調べて、以下のレコードを登録してください。

| ホスト名         | タイプ | TTL   | 値                                 |
| --------------- | ------| ----- | ---------------------------------- |
| ns0-idaas.procube-demo.jp |   A   | 1時間  | hive0.idaasのElastic IP |
| ns1-idaas.procube-demo.jp |   A   | 1時間  | hive1.idaasのElastic IP |
| ns2-idaas.procube-demo.jp |   A   | 1時間  | hive2.idaasのElastic IP |


### 6. hive.yml ファイルの編集
hive.yml ファイルを自分の環境に合わせて、修正してください。
hive-builder で構築コードを作成するときの hive名となる文字列を決定していただく必要があります。
VPC名にもなりますので、AWSコンソールから見たときに一覧に表示されることになります。
以下に hive名の埋め込み対象を示します。

| 適用対象          | 生成ルール        |例                |
| --------------- | ---------------- | ---------------- |
|VPC名             | vpc-${HIVENAME} | vpc-hive-ho0 |
|internet gateway の Name タグ| gateway for vpc-${HIVENAME} | geteway for vpc-hive-ho0 |
|security group の Name タグ| group-vpc-${HIVENAME} | group-vpc-hive-ho0 |
|Project タグ| ${HIVENAME} | hive-ho0 |
|long hostname|s-hive[0-3].${HIVENAME} | s-hive0.hive-ho0 |
|ゾーンのFQDN|${HIVENAME}.procube.info| hive-ho0.procube.info|

hive名が決定したら hive.yml の name 属性に設定してください。
また、AWS では、ルートアカウントによって、利用できる可用性ゾーンが異なります。
hive.yml の production ステージオブジェクトの subnet 属性のサブネットの配列で available_zone 属性を適宜修正してください。

### 7. 構築
以下の手順で hive を構築してください。

    hive all

このコマンドは80分程度かかります。

### 8. サーバ証明書の取得
以下のコマンドを実行してサーバ証明書を取得します。

    hive ssh
    dsh webgate
    ansible-playbook -i /var/acme/hosts acme.yml

このコマンドでは、テスト用のサーバ証明書が発行されます。エラーが発生した場合は、手順を中断して質問してください。
このシェルを開いたまま https://idm.idaas.procube-demo.jp にアクセスしてサーバ証明書が信頼できない旨の警告が出ればOKです。
次に本物のサーバ証明書を取得するために /var/acme/acme-vars.yml を vi で開き、最初の行を "#" でコメントアウトし、次の行のコメントアウトを外してください。
その後、上記のシェルの続きで次のコマンドを実行してください。


    rm -f /var/acme/data/*
    ansible-playbook -i /var/acme/hosts acme.yml
    exit
    exit
    

### 9. 動作確認
ホスト側のブラウザで https://idm.idaas.procube-demo.jp にアクセスし、以下のアカウントでログインし、IDaaSメタID管理システムが表示されれば成功です。

| 項目       | 値                                                                 |
| ---------- | ----------------------------------------------------------------- |
| ユーザID   | administrator@idaas.procube-demo.jp                                         | 
| パスワード  | YVKvZzz4z5NGrpV1A (.hive/production/registry_password の値の後ろに"1A"を追加したもの)|

ログイン後、最初の１回は shibsp::ListenerException のエラーが発生する場合がありますが、その場合はリロードしてください。
フォームデータの再送信に対する警告が表示される場合もありますが、再送信を実行してください。

### 10. データのアップロード
各種初期データをアップロードします。

### 10-1. テナントのアップロード
前項のIDaaSメタID管理システムにログインした状態で、「ID管理」→「テナント管理」のタブで「CSVアップロード」ボタンをクリックし、以下のCSVデータをアップロードしてください。

    ,tenantid,displayName,amid,authMethod,lifetime,inactivity,usePeriodic,daysForExpire,saml_sps
    Create,test1,テスト1,am0,Password,16,8,FALSE,,
    Create,test2,テスト２,am1,MFA,30,20,TRUE,8,
    Create,test3,テスト3,am2,Password,10,3,FALSE,,
    Create,test4,テスト4,am1,MFA,12,12,TRUE,8,
    Create,test5,テスト5,am2,Password,12,12,FALSE,,
    Create,test6,テスト6,am2,Password,12,12,FALSE,,

「プロビジョニング発効確認」画面で「保留」をクリックしてください。

### 10-2. speedtest メタデータの登録
以下の手順で test1 テナントに speedtest サービスのメタデータを登録してください。

1. https://speedtest.idaas.procube-demo.jp/Shibboleth.sso/Metadata にアクセスして speedtest アプリのメタデータをダウンロードしてください。最初の１回は「Metadata Request Failed」と表示され、エラーになる場合もありますが、その場合はリロードしてダウンロードしてください。
1. 前項のIDaaSメタID管理システムのID管理」→「テナント管理」のタブに戻り、test1 の行をクリックして詳細画面を開いてください。
1. 「更新」をクリックして、編集画面を開き、SAML SP リストの「追加」ボタンをクリックしてください。
1. 「公開名」に「speedtest」と入力し、「SPメタデータ」に 1. でダウンロードしたファイルに含まれているXMLを貼り付けてください。
1. 「SAMLアサーションに署名する」をチェックしてください。
1. 「保存」ボタンで前の画面に戻り、もう一度「保存」ボタンをクリックしてください
1. 「プロビジョニング発効確認」画面で「発効」をクリックしてください。

### 10-3. 利用者の登録
IDaaSメタID管理システムの「ID管理」→「利用者管理」のタブで「CSVアップロード」ボタンをクリックし、以下のCSVデータをアップロードしてください。

    ,tenantid,uid,cn,idmRole,userPassword,dn,accountLock,passwdChangedDatetime,expireDate,data1,data2,data3,data4,data5
    Create,test1,adm1@procube.jp,管理者１太郎,IDM_USER_ADMIN,ZrIxphyVvBSrdL11A,"uid=adm1@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test2,adm2@procube.jp,管理者２次郎,IDM_USER_ADMIN,ZrIxphyVvBSrdL11A,"uid=adm2@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test3,adm3@procube.jp,管理者３太郎,IDM_USER_ADMIN,ZrIxphyVvBSrdL11A,"uid=adm3@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test4,adm4@procube.jp,管理者４次郎,IDM_USER_ADMIN,ZrIxphyVvBSrdL11A,"uid=adm4@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test5,adm5@procube.jp,管理者５太郎,IDM_USER_ADMIN,ZrIxphyVvBSrdL11A,"uid=adm5@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test6,adm6@procube.jp,管理者６次郎,IDM_USER_ADMIN,ZrIxphyVvBSrdL11A,"uid=adm6@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test1,user1@procube.jp,利用者１太郎,IDM_USER,ZrIxphyVvBSrdL11A,"uid=user1@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test2,user2@procube.jp,利用者２次郎,IDM_USER,ZrIxphyVvBSrdL11A,"uid=user2@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test3,user3@procube.jp,利用者３太郎,IDM_USER,ZrIxphyVvBSrdL11A,"uid=user3@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test4,user4@procube.jp,利用者４次郎,IDM_USER,ZrIxphyVvBSrdL11A,"uid=user4@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test5,user5@procube.jp,利用者５太郎,IDM_USER,ZrIxphyVvBSrdL11A,"uid=user5@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,
    Create,test6,user6@procube.jp,利用者６次郎,IDM_USER,ZrIxphyVvBSrdL11A,"uid=user6@procube.jp,ou=Users,DC=idaas,DC=procube-demo,DC=jp",FALSE,,,,,,,

「プロビジョニング発効確認」画面で「発効」をクリックしてください。

### 10-4. プロビジョニングタスクの完了確認
IDaaSメタID管理システムの「プロビジョニング管理」→「プロビジョニングタスク閲覧」のタブで一覧表示の一番左の列に「無視」のボタンがあることを確認してください。
これは、11-2. のプロビジョニング要求に対するタスクが実行中であることを示しています。
11-2 の作業実施後（実施時刻は「登録日時」の列に表示されています）、12分程度経過したところで「最新表示」ボタンをクリックして「無視」のボタンが消えることを確認してください。

### 10-5. テナントユーザの動作確認
ブラウザのシークレットウィンドウ/プライベートウィンドウを新しく開いて、https://speedtest.idaas.procube-demo.jp/Shibboleth.sso/Login?entityID=https%3A%2F%2Fam0.idaas.procube-demo.jp%2Ftest1%2Fshibboleth にアクセスし、以下のIDでログインしてください。

| 項目       | 値                  |
| --------- | ------------------- |
| ユーザID   | user1@procube.jp    | 
| パスワード  | ZrIxphyVvBSrdL11A   |

speedtest のアプリケーションが表示されればOKです。

### 11. zabbix の通知設定

zabbix から Google chat への通知を設定します。

### 11-1. zabbix へのログイン

http://monitor.idaas.procube-demo.jp を開いて、以下の認証情報でログインしてください。

| 項目        | 値            |
| ---------- | ------------- |
| ユーザID    | admin         | 
| パスワード   | zabbix        | 

### 11-2. admin ユーザの日本語設定

左メニューの Administration → Users を選択し、ユーザ一覧を表示し、 Admin をクリックして設定画面を開いてください。
Language に Japanese(ja_JP) を選択し、 Timezone に （UTC＋09:00)Asia/Tokyo を選択し、Update ボタンを押してください。Timezone のドロップダウンリストは長くて選択しにくいです。UTCとの時間差順になっているので、+09:00 はかなり下の方にあることに注意してください。

### 11-3. メディアタイプの設定

左メニューの 「管理」 → 「メディアタイプ」 を選択し、メディアタイプ一覧を表示し、右上の「インポート」ボタンを押して、インポート画面を開いてください。
「ファイルを選択」ボタンを押して、zbx_googlechat.yml を選択して「インポート」ボタンを押してください。
左メニューの 「管理」 → 「メディアタイプ」 を選択し、メディアタイプ一覧を表示し、GoogleChat をクリックして、設定画面を開いてください。「chat_url」 Google chat のスペースの WebHook のURLをコピペしてください。
 Google chat のスペースの WebHook のURLは  Google chat のスペースの画面のスペース名の右の▼をクリックして「Webhookを管理」を選択して、着信Webhook を追加し、そのURLをコピペしてください。
 その後、「更新」ボタンを押してください。

### 11-4. admin ユーザのメディアタイプの設定

左メニューの 「管理」 → 「ユーザー」 を選択し、ユーザー一覧を表示し、Admin をクリックして設定画面を開いてください。
開いた画面の「メディア」タブをクリックし、メディアの「追加」リンクをクリックしてください。
タイプに「GoogleChat」を選択し、送信先にスペース名を入れてください。
「追加」ボタンをクリックし、メディアタブの画面で「更新」ボタンを押しcてください。
「追加」だけでは変更が反映されないので、注意してください。

### 11-5. アクションの有効化

左メニューの 「設定」 → 「アクション」 を選択し、アクション一覧を表示し、Report problems to Zabbix administrators をクリックして設定画面を開いてください。「有効」チェックして、「更新」ボタンを押してください。

### 12. データのリストア

検証環境などからデータをリストアする場合は、以下の手順に従ってください。

#### 12-1. データの採取

採取元のマザー環境で以下のコマンドを実行し、バックアップを取得してください。
（リポジトリサーバ名：hive3.idaas 、およびコンテキストディレクトリ：.hive/staging は、採取元の環境にあわせて読み替えてください。）

    hive ssh
    tar cvzhf backup.tar.gz backup/*-latest.*
    exit
    scp -F .hive/staging/ssh_config hive3.idaas:backup.tar.gz .

採取元マザー環境からさらに scp などで backup.tar.gz を手元のマザー環境にダウンロードしてください。

#### 12-2. データのリストア

マザー環境で以下のコマンドを実行し、データをリストアしてください。

    scp -F .hive/production/ssh_config backup.tar.gz hive3.idaas:
    hive ssh
    mv backup backup.org # まだバックアップが一度も実行されていない場合は backup ディレクトリが無いため実行不要です。
    tar xvzf backup.tar.gz backup
    hive-backup.sh -r
    exit

### 13. 停止と起動

#### 停止
全サーバを停止する場合は以下のコマンドを実行してくだいさい。

    hive build-infra -H

#### 起動

全サーバを起動する場合は以下のコマンドを実行してくだいさい。

    hive build-infra

### 14. 破棄

サイト全体を破棄する場合は以下のコマンドを実行してくだいさい。

    hive build-infra -D
