

# isucon11-qualifyの環境構築
実際のisucon11予選
https://github.com/isucon/isucon11-qualify/blob/main/docs/manual.md

CloudFormationのyml
https://github.com/isucon/isucon11-qualify/tree/main/provisioning/cf-kakomon

iamを差し替える
https://github.com/matsuu/aws-isucon

CloudFormationからアップロード
isucon11-qualify.yml

EC2キーペアを作成してログイン
sudo ssh -i ~/Desktop/isucon11-qualify-key.pem ubuntu@18.180.200.210

各サーバーにsshして3人の公開鍵登録
```
isucon11-qualify-1 18.179.238.18
isucon11-qualify-2 18.180.189.225
isucon11-qualify-3 52.192.133.236
isucon11-bench 18.180.200.210
```


## isucon準備

アプリの重要箇所っぽいところにはログを仕込む
- 特定のユーザー数人（Cookieから追跡）がどういう順でエンドポイントにアクセスしているか
- とあるエンドポイントで何本のSQLが発行されているか


便利なツールを入れてみる
netdata
new relic


再起動スクリプトを作っておく
MySQLの再起動。Nginxの再起動。


## 最初にすること
- とりあえずベンチを実行
- git管理

```
git init
git add . && git commit -m "initial state"

nginxやMySQLの設定もリポジトリ内にコピー＆シンボリックリンクを置く
```


ログを仕込んで、その後計測
- CPU専有率の高いプロセスがないか
- アクセスログ
- DBの発行クエリ
- メモリ
- ネットワーク帯域
- ディスクIO




## サーバー

サーバーの状態を確認

```
# CPU
cat /proc/cpuinfo

# メモリ
free -h

# サービス
ls -la /usr/lib/systemd/system/
systemctl list-units --type=service --state=running

# CPU専有率の高いプロセスがないか
top
htop
```


## Nginx
--------------------
- アクセスログで見るべきは`avg`と`count`
- 304で返す
- ログを仕込んでベンチ回した後、どのURLがポイント高いか目星をつけた方が良さそう

```bash
cat /etc/nginx/nginx.conf
ls -la /etc/nginx/conf.d

cat /var/log/nginx/access.log | alp ltsv --sort count -r
cat /var/log/nginx/access.log | alp ltsv --sort avg -r

echo '' > access.log && echo '' > error.log

# 起動確認
nginx -t

sudo systemctl status nginx.service
sudo systemctl restart nginx.service
```


## MySQL
--------------------
- スロークエリ、発行クエリのログを設定
- 見るべきは発行回数と平均実行時間
- pt-query-digestとかもある
- DB・テーブルのデータ量をみてキャッシュサイズ決めた方が良さそう

```
# 設定ファイル
cat /etc/my.cnf
cat /etc/mysql/my.cnf
cat /etc/mysql/conf.d/my.cnf

sudo systemctl status mysql.service
sudo systemctl restart mysql.service
```

my.cnf オプション参考
https://dev.mysql.com/doc/refman/5.6/ja/server-option-variable-reference.html

```sql
ALTER TABLE isubata.image ADD created_at datetime AFTER data;
ALTER TABLE isubata.image DROP created_at;
```

## PHP
--------------------
- APCu使うとよさそう

```
php-fpm -ini | grep fpm
ls -la /usr/local/etc/php/conf.d

php -i | grep ini
cat /usr/local/etc/php/conf.d/php.ini

php -f xxx.php

# php周り
cd /usr/local/etc
cat /usr/local/etc/php-fpm.d/www.conf
```


OPcacheの有効化
/usr/local/etc/php/conf.d/php.ini
```
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=4000
opcache.revalidate_freq=60
opcache.fast_shutdown=1
opcache.enable_cli=1
```




## 後始末
- MySQLやアプリのログを外す
- MySQLを再起動しておく
- New Relic外す



dc down && dc build && dc up -d
# 1分くらい待ってから
dc up bench


for i in {1..5} ; do dc up bench ; done
