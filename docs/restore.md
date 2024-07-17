# バックアップファイルをリストアする方法

hive-builder で構築したリポジトリサーバでは、データの論理バックアップが採取されます。
深夜1:00に動作する日次バッチでサービス定義の backup_scripts 属性に指定された内容でリポジトリサーバの ~/backup ディレクトリにバックアップが採取されます。
同時にリポジトリと zabbix データベースの内容もバックアップされます。
また、手動で  hive-backup.sh でバックアップを採取することが可能です。

この文書ではバックアップファイルの運用手順を示します。文書中に出現する idaas については、自身の hive名に置き換えてください。

## サイトを再構築する

ここでは、例としてすべてのバックアップファイルを mother ホストに一時的に退避して、サイトを再構築にリストアする手順を示します。

### 1. mother ホストへのバックアップファイルの退避

以下のコマンドを実行して、バックアップを mother ホストに退避してください。

```
hive ssh
hive-backup.sh
tar cvzhf backup.tar.gz backup/*-latest.*
exit
scp -F .hive/production/ssh_config hive3.idaas:backup.tar.gz .hive/production/backup.tar.gz
```

ただし、 build-images を繰り返している場合はリポジトリサーバが想定以上に大きくなっている場合があります。

```
hive ssh
cd registry
docker exec registry du -sh /var/lib/registry
```

で、ディレクトリのサイズが 5G 以上あってサイズが大きい場合は以下のコマンドでリポジトリサーバのバックアップファイルを小さくできます。
通常、 hive all 直後のディレクトリのサイズは 3.5G 程度です。

```
docker-compose down -v
docker-compose up
docker image ls --format '{{ .Repository }}:{{ .Tag }}' | grep hive3.idaas | xargs -L 1 docker image push
```

### 2. 再構築

以下のコマンドでサイトを再構築してください。

```
hive build-infra -D
hive all
```

### 3. データのリストア

以下のコマンドでデータをリストアしてください。

```
scp -F .hive/production/ssh_config .hive/production/backup.tar.gz hive3.idaas:
hive ssh
tar xvzf backup.tar.gz
hive-backup.sh -r
exit
```

### 4. サービスの再デプロイ

以下のコマンドで全サービスを再デプロイしてください。

```
hive deploy-services -D
hive deploy-services
```

デプロイ後、WebGate, AccessManager が起動するのに 2,3 分かかります。
テストはその後に実施してください。
