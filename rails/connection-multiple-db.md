# Railsでの複数DBの接続方法

大きく分けてやることが３つある

- database.ymlに新しい接続先を設定
- 接続用の共通モデルと別DBテーブルごとのモデルを作成
- 別DB操作用のtaskを作成

## database.ymlに新しい接続先を設定
まず、別DB用にdatabase.ymlに新しい接続先を設定する。

```database.yml
default: &default
  adapter: mysql2
  encoding: utf8
  username: root
  password:
  host: localhost
  pool: 5
  timeout: 5000

another_db: &another_db
  adapter: mysql2
  encoding: utf8
  username: root
  password:
  host: localhost
  pool: 5
  timeout: 5000

development: &development
  <<: *default
  database: rails_development

development_another_db:
  <<: *another_db
  database: rails_another_db_development

test: &test
  <<: *default
  database: rails_test

test_another_db:
  <<: *another_db
  database: rails_another_db_test
```

## 接続用の共通モデルと別DBテーブルごとのモデルを作成
次に先ほど作成した接続先に繋ぐためのモデルを作成する。

```another_db.rb
class AnotherDB < ActiveRecord::Base
  # 接続先の指定
  establish_connection("#{Rails.env}_another_db".to_sym)

  # 接続用の抽象モデルということを指定
  # これを指定しないと実際に使うモデルとして認識し、テーブルを探しに行こうとする
  self.abstract_class = true
end
```

作成した共通モデルを元として実際に使うモデルを作成する。

```user.rb
class User < AnotherDB
end
```

共通モデルを作成せずに一つ一つのモデルで `establish_connection` するという方法もあるが、
これをすると各モデルを使う歳に個々でDB接続のセッションを張りに行こうとするので、共通モデルを作るのがベター。
参考：[ActiveRecordのestablish_connectionに気をつけろ](http://d.hatena.ne.jp/raugisu/20120428/1335598633)

## 別DB操作用のtaskを作成
普通に `rake db:create` などすると一つ目のDBに繋ごうとするので、別でtaskを作ってあげる。

```another_db.rake
namespace :another_db do
  task :create do
    # database.ymlの別DB接続設定の読み込み
    CONF = YAML.load(File.open(File.join('config/database.yml')))["#{Rails.env}_another_db"]
    ActiveRecord::Base.establish_connection CONF.merge(database: 'mysql')
    ActiveRecord::Base.connection.create_database CONF['database']
  end

  task :migrate do
    CONF = YAML.load(File.open(File.join('config/database.yml')))["#{Rails.env}_another_db"]
    ActiveRecord::Base.establish_connection CONF

    # migrateを行う先を指定する
    # 別DBのプロジェクトがあれば、そちらをtmpに置いて指定する
    ActiveRecord::Migrator.migrate('tmp/another_db_project/db/migrate/')
  end

  task seed: :environment do
    CONF = YAML.load(File.expand_path(File.open(File.join('config/database.yml'))))["#{Rails.env}_another_db"]
    ActiveRecord::Base.establish_connection CONF

    # Seed-Fuを使う場合はこのようにする
    # seedファイルはdevelopment_another_db下に置く感じで
    fixture_paths = ['db/fixtures/' + "#{Rails.env}_another_db"]
    SeedFu.seed(fixture_paths)
  end

  task :rollback do
    CONF = YAML.load(File.open(File.join('config/database.yml')))["#{Rails.env}_another_db"]
    ActiveRecord::Base.establish_connection CONF
    ActiveRecord::Migrator.rollback('tmp/another_db_project/db/migrate/')
  end
end
```
