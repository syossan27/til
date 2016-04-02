# websocket-railsの使い方

基本的には公式に書いてある通りだがハマりどころを何点か。
※今回はUnicornを使用

## Redisの接続先を設定する

config/initializers/redis.rbを作成する

```
$redis = Redis.new(host: 'localhost', port: 6379, driver: :hiredis)
```

※しないとこうなる
![error1](https://github.com/syossan27/til/blob/master/image/rails_error_1.png)

## EventMachineの設定

非スタンドアローンモードの場合、config/initializers/eventmachine.rbを作成する

```
Thread.new { EventMachine.run } unless EventMachine.reactor_running? && EventMachine.reactor_thread.alive?
```

※スタンドアローンモードの場合だと逆にスタンドアローンサーバが立ち上がらなくなるので注意

## 依存ライブラリであるfaye-websocketの特定バージョンを指定する

0.10.0で指定する。こうしないとwebsocketの読込がpending状態のままになる。

```Gemfile
gem 'faye-websocket', '0.10.0'
```

## Redisを使用する場合のwebsocket設定

非スタンドアローンモードの場合、websocket_rails.rbを以下のように設定

```
config.standalone = false
config.synclonize = true
config.redis_options = {:host => 'localhost', :port => '6379', :driver => :hiredis}
```

スタンドアローンモードの場合、websocket_rails.rbを以下のように設定

```
config.standalone = true
config.synclonize = false
config.redis_options = {:host => 'localhost', :port => '6379', :driver => :hiredis}
```

## JS Clientの接続設定

```
dispatcher = new WebSocketRails('localhost:3000/websocket')
```

## その他

今回の場合、スタンドアローンサーバを立ち上げた状態、かつ接続先を3001ではなく3000にしなければwebsocketが使えるようにならなかった。

3001に接続した場合、以下のエラーメッセージが表示された。

```
WebSocket connection to 'ws://localhost:3001/websocket' failed: Error in connection establishment: net::ERR_CONNECTION_REFUSED
```

また、スタンドアローンサーバを立ち上げなかった場合は3000ではエラーは出ないのだがイベントルータを通ってコントローラーまで処理が到達しなかった。
