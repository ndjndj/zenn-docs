---
title: "Rails 過去の migration を転生(down→修正→up)させる"
emoji: "🐰"
type: "tech"
topics:
  - "rails"
published: true
published_at: "2024-02-14 13:05"
---

Rails で API を開発しています。
作成したテーブルのカラムを削除したかったのですが、テストデータも入れておらず、開発も一人でしているので、マイグレーションファイルを新規作成せずに、作成したテーブルを削除 → スキーマを修正したテーブルを新規テーブルとして作成することにしました。

# migration ファイルを up と down に分ける
元のマイグレーションファイルは change メソッドのみが定義されていた。
今回は、before のマイグレーションファイル+`rails db:migrate` によって、テーブルが作成されていた。という状況。

**before**
```ruby
# db/migrate/xxxxxxxxxxxxx_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.1]
  def change
    create_table :users, primary_key: [:uuid] do |t|
      t.string :uuid, :null => false
      t.string :name, limit: 100
      
      t.timestamps
    end
  end
end
```

このマイグレーションファイルに up メソッドと、down メソッドを定義しなおす。
今回、down メソッドでは、テーブルを削除する処理をおこなう。
up メソッドでは、先ほどのテーブルに email カラムを追加している。

**after**
```ruby
# db/migrate/xxxxxxxxxxxxx_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.1]
  def up
    create_table :users, primary_key: [:uuid] do |t|
      t.string :uuid, :null => false
      t.string :name, limit: 100
      t.string :email, limit: 100
      
      t.timestamps
    end
  end

  def down
    drop_table :users
  end
end
```

:::message 
before のままでも、`rails db:migrate:down VERSION=xxxx` でロールバックすることも可能らしいです。

https://qiita.com/pyon_kiti_jp/items/a23660d20e76fffa5dd4
:::

# 一旦テーブルを削除する

## Migration ID の特定
なかったことにしたいマイグレーションを特定する。

```bash
rails db:migrate:status
```

下記のようなレスポンスが返ってくる。
Users テーブルを一度削除したいので今回は、**20240130121500**が対象のマイグレーション ID ということがわかった。

```
 Status   Migration ID    Migration Name
--------------------------------------------------
   up     20231219085821  Create buzz
   up     20231219095909  Create bar
   up     20240130121500  Create users
   up     20240208100803  Create foo
```

## 削除の実行
先ほど定義した down メソッドを実行し、Users テーブルを drop する。

```
rails db:migrate:down VERSION=20240130121500
```

# 再度テーブルを作成する
修正した up メソッドを実行し、Users テーブルを新規作成する。

```
rails db:migrate:up VERSION=20240130121500
```

schema.rb などで、目的どおりのものが作成されているかどうか確認する

# さいごに
すでにデータが入っちゃているとか一人でやっていないとか、実稼働しているとかなら絶対に migration ファイルを作成したほうがいいというかどんなときも、migration ファイルを作成すべきなのでしょう。

# 参考
https://railsdoc.com/migration
https://qiita.com/pyon_kiti_jp/items/a23660d20e76fffa5dd4