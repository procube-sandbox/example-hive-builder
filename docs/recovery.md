# サーバをイメージバックアップから復旧する方法

ここでは、EC2インスタンスが完全に失われた場合に、AMIから復旧する手順を示します。

## 0. バックアップの採取

各サーバごとにAMI を作成して、バックアップを採取してください。
どのサーバのイメージかわかるように AMI に名前をつけておいてください。

## 1. hive0 を demote

生き残っているサーバにログインして hive0 を worker に demote してください。

```
hive ssh -t hive1.${hivename}
docker node demote hive0
exit
```

## 2.  EC2 インスタンスの起動
AMI からインスタンスを起動してください。以下のパラメータを入力してください。

* タグに Name=hive0.${hivename}, Project=${hivename} を追加
* インスタンスタイプを t3.largeに設定
* キーペアに hive-builder-${hivename} を選択
* ネットワークを vpc-${hivename} に変更
* サブネットを subnet-a に設定
* 既存のセキュリティグループを選択
* 高度なネットワーク設定→プライマリIPに 192.168.0.5 （hive0 のプライベートIP）を入力

## 3. Elastic IP の紐付け

hive0 の Elastic IP が残っていれば作成したインスタンスに関連付けてください。残っていない場合は新しく作成して関連付けてください。新しく作成して関連付けた場合は、 .hive/production/ssh_config にかかれている hive0 のグローバルIPを新しいものに変更してください。

## 4. ssh ホストキーの設定

ssh のホストキーが変わっているため、以下の手順で ssh を設定してください。
1. .hive/production/known_hosts から hive0 の Elastic IP に対応する行を削除
2. .hive/production/ssh_config の hive0 の StrictHostKeyChecking の値を ask に変更
3. hive ssh -t hive0.${hivename} でログイン（ホストキーの正当性を聞かれるので yes を入力）してログアウト
4. .hive/production/ssh_config の hive0 の StrictHostKeyChecking の値を yes に変更

## 5. 動作確認

hive0 にログインして動作を確認してください。

```
hive ssh -t hive0.${hivename}
docker info # docker デーモンが動作していて swarm が active になっていることを確認
drbdadm status # エラーになっているボリュームがないことを確認
exit
```

drbd のステータスが Inconsistent になっている場合で SyncTarget が表示されているものがある場合は、その表示がなくなるまでしばらく
待ってください。数十分かかる場合があります。


## 6. hive0 を promote

生き残っていたサーバにログインして hive0 を manager に promote してください。

```
hive ssh -t hive1.${hivename}
docker node promote hive0
exit
```

## 7. 以降の起動

復旧以降は EC2インスタンスの起動元のイメージIDが変わるため、```hive build-infra``` コマンドによる起動はできません。AWSコンソールから起動するようにしてください。
