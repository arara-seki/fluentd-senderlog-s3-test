# 手順

## 1. 環境変数を設定する

```
cp ./.env.example ./.env
vi ./.env
```

## 2. Docker イメージをビルドする

```
docker build -t myfluentd .
```

## 3. Docker コンテナを起動する

```
docker run --rm \
    --env-file=.env \
    -v $(pwd)/data:/fluentd/data \
    -v $(pwd)/conf:/fluentd/etc \
    myfluentd
```

## 4. ログファイルにテキストを追記する

`(date<tab>user_id<tab>message)`

```
echo -e "2020/12/01 22:00:00\ttestuser\thello" >> ./data/send.log
```

## 5. fluent.conf で読み込んでいる設定ファイルを切り替える

`./conf/fluent.conf`
```
#@include a.conf
@include b.conf
```

# Fluentd Event データ詳細

シナリオ: 以下のテキストログを 2020-12-01T09:00:00+09:00 にログに書き込むことを想定

```
echo -e "2020/12/01 08:59:59\ttestuser\thello" >> ./data/send.log
```

## a.conf

|plugin|time|tag|record|
|:--:|:--:|:--:|:--:|
|tail|2020-12-01 09:00:00 +0000|send_log|{"time":"2020/12/01 08:59:59","user_id":"testuser","message":"hello"}|
|record_transformer|2020-12-01 09:00:00 +0000|send_log|{"time":"2020-11-30 23:59:59","user_id":"testuser","message":"hello"}|
|s3|2020-12-01 09:00:00 +0000|send_log|{"time":"2020-11-30 23:59:59","user_id":"testuser","message":"hello"}|
|S3オブジェクト|dt=20201201|nil|{"time":"2020-11-30 23:59:59","user_id":"testuser","message":"hello"}|

`time` には tail プラグインで処理した時刻がJST(local time)で記録される。  
これが S3 オブジェクトの path に出力されるため、ログに出力されている時刻とパーティションが異なってしまう可能性がある。

## b.conf

|plugin|time|tag|record|
|:--:|:--:|:--:|:--:|
|tail|2020-11-30 23:59:59 +0000|send_log|{"user_id":"testuser","message":"hello"}|
|s3|2020-11-30 23:59:59 +0000|send_log|{"user_id":"testuser","message":"hello","time":"2020-11-30 23:59:59"}|
|S3オブジェクト|dt=20201130|nil|{"user_id":"testuser","message":"hello","time":"2020-11-30 23:59:59"}|

`time` には tail プラグインで処理したtsvレコードの中身が反映される。  
`timezone` を `+09:00` に指定することで、UTCでの値で保持される。  
S3 プラグインの `inject` セクションで `time` の値をレコード内に差し込むことで JSON ログにもUTCの時刻を出力する。