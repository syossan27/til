# Pumaの設定ファイル例の日本語説明板

自分用に日本語翻訳しました。

原文：https://github.com/puma/puma/blob/master/examples/config.rb

```
#!/usr/bin/env puma

# pumaが動作するディレクトリを指定します。
#
# デフォルトはカレントディレクトリになります。
#
# directory '/u/apps/lolcat'

# Rackアプリケーションのブロックやオブジェクトを使用する。
# この設定ファイルはアプリケーション自身とすることができます。
#
# app do |env|
#   puts env
#
#   body = 'Hello, World!'
#
#   [200, { 'Content-Type' => 'text/plain', 'Content-Length' => body.length.to_s }, [body]]
# end

# Rackup fileとして"path"をロードします。
#
# デフォルトは"config.ru"です。
#
# rackup '/u/apps/lolcat/config.ru'

# Rackアプリケーションを実行する環境を設定します。
# 値は文字列とします。
#
# デフォルトは"development"です。
#
# environment 'production'

# サーバをデーモンとして起動し、バックグランド処理させます。
# "pidfile"と"stdout_redirect"を合わせて設定するのをオススメします。
#
# デフォルトは"false"です。
#
# daemonize
# daemonize false

# 設定した"path"にサーバのpidをファイルとして格納します。
#
# pidfile '/u/apps/lolcat/tmp/pids/puma.pid'

# 設定した"path"にサーバ状態の情報をファイルとして格納します。
# このファイルは"pumactl"のクエリ実行や、サーバのコントロールに使われます。
#
# state_path '/u/apps/lolcat/tmp/pids/puma.state'

# 指定したファイルにSTDOUTとSTDERRをリダイレクトさせます。
# ３つ目のパラメータ（"append"） は出力が追加記載されるかどうかを指定します。
# デフォルトは"false"です。
#
# stdout_redirect '/u/apps/lolcat/log/stdout', '/u/apps/lolcat/log/stderr'
# stdout_redirect '/u/apps/lolcat/log/stdout', '/u/apps/lolcat/log/stderr', true

# リクエストログの有効化設定
#
# デフォルトは"false"です。
#
# quiet

# リクエストに応答するためのスレッド数を設定します。
# "min"は最小スレッド数、"max"は最大スレッド数です。
#
# デフォルトは"0, 16"です。（最初の数値が"min"、最後の数値が"max"）
#
# threads 0, 16

# Bind the server to "url". "tcp://", "unix://" and "ssl://" are the only accepted protocols.
# 設定したプロトコルのみサーバにバインドさせます。
# 設定可能なのは"url". "tcp://", "unix://", "ssl://"の４つです。
#
# デフォルトは"tcp://0.0.0.0:9292"です。
#
# bind 'tcp://0.0.0.0:9292'
# bind 'unix:///var/run/puma.sock'
# bind 'unix:///var/run/puma.sock?umask=0111'
# bind 'ssl://127.0.0.1:9292?key=path_to_key&cert=path_to_cert'

# Instead of "bind 'ssl://127.0.0.1:9292?key=path_to_key&cert=path_to_cert'" you can also use the "ssl_bind" option.
# "bind 'ssl://127.0.0.1:9292?key=path_to_key&cert=path_to_cert'"といったバインド設定の代わりに、
# "ssl_bind"といったオプションが使用できます。
#
# ssl_bind '127.0.0.1', '9292', {
#   key: path_to_key,
#   cert: path_to_cert
# }
# JRubyでは追加のキーが必須です。
# keystore: path_to_keystore,
# keystore_pass: password

# 再起動させる前にコードを実行します。
# このコードはログファイル、データベース接続、その他諸々をクローズさせるためにあります。
#
# on_restart do
#   puts 'On restart...'
# end

# Pumaを再起動させるためのコマンドを設定します。
# This should be just how to load puma itself (ie. 'ruby -Ilib bin/puma'), not the arguments to puma, as those are the same as the original process.
#
# restart_command '/u/app/lolcat/bin/restart_puma'

# === クラスターモード ===

# ワーカープロセス数の設定
#
# デフォルトは0
#
# workers 2

# ワーカーが立ち上がる前に実行されるコードの設定
#
# before_fork do
#   puts "Starting workers..."
# end

# リクエストを受ける前にワーカー上で実行されるコードの設定
#
# これはワーカーが立ち上がる度に呼び出されます。
#
# on_worker_boot do
#   puts 'On worker boot...'
# end

# 終了前にワーカー上で実行されるコードの設定
#
# ワーカーがシャットダウンされる前に呼び出されます。
#
# on_worker_shutdown do
#   puts 'On worker shutdown...'
# end

# ワーカーが立ち上がる前に実行されるコードの設定
# ワーカーのindexが引数として渡されます。
#
# これはワーカーが立ち上がる度に呼び出されます。
#
# on_worker_fork do
#   puts 'Before worker fork...'
# end

# ワーカーが立ち上がった後に実行されるコードの設定
# ワーカーのindexが引数として渡されます。
#
# これはワーカーが立ち上がる度に呼び出されます。
#
# after_worker_fork do
#   puts 'After worker fork...'
# end

# マスタープロセスがUSR1シグナルを発行した時にワーカーがバンドルコンテキストをリロードするかどうかの設定
# マスターがphased-restartしてもGemの適切な再読み込みを可能にします（preload_appと互換性はありません）
#
# デフォルトはOFFです。

# prune_bundler

# ワーカーを開始する前にアプリケーションをプリロードする。
# これはphased-restartの機能とコンフリクトします。（デフォルトはOFF）

# preload_app!

# Process listing内で追加テキストを表示させます。
#
# tag 'app name'
#
# タグを指定しない場合はPumaが推測します。
# Pumaの推測によるタグ追加をしたくない場合は、空の文字列を設定します。

# 全てのワーカーが指定したタイムアウト設定時間内にマスタープロセスにチェックインしているかを確認します。
# 時間内にチェックしていない場合、ワーカープロセスが再起動されます。
# デフォルトは60秒です。
#
# worker_timeout 60

# ワーカーの起動時のタイムアウト時間を設定します。
#
# 指定しない場合はデフォルトの `worker_timeout` が設定されます。
# デフォルトは60秒です。
#
# worker_boot_timeout 60
```
