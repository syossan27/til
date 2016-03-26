# websocket-railsの使い方

基本的には公式に書いてある通りだがハマりどころを何点か。

## Redisの接続先を設定する

config/initializers/redis.rbを作成する

```
$redis = Redis.new(host: 'localhost', port: 6379, driver: :hiredis)
```

## EventMachineの設定

config/initializers/eventmachine.rbを作成する

```
Thread.new { EventMachine.run } unless EventMachine.reactor_running? && EventMachine.reactor_thread.alive?
```

## 依存ライブラリであるfaye-websocketの特定バージョンを指定する

0.10.0で指定する。こうしないとwebsocketの読込がpending状態のままになる。

```Gemfile
gem 'faye-websocket', '0.10.0'
```

## Redisを使用する場合のwebsocket設定

websocket_rails.rbを以下のように設定

```
config.standalone = false
config.synclonize = true
config.redis_options = {:host => 'localhost', :port => '6379', :driver => :hiredis}
```
