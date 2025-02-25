---
title: "Rails ActiveRecord で 'WHERE X=x OR Y=y' したい"
emoji: "🐰"
type: "tech"
topics:
  - "rails"
  - "ruby"
  - "activerecord"
published: true
published_at: "2024-06-10 20:41"
---

表題のように、`SELECT * FROM TABLE_A WHERE id = 1 OR name IS NULL` みたいなことをしたかったのでやり方を調べました
基礎的な内容ですが、where とは使い勝手が違う感じがしたので、備忘録としてまとめます。


# やりかた
至極単純です。
特定の条件で where した後に `or()` メソッドをチェーンしてその中に別の条件を記すだけです。

```rb
Review.where(id: 1).or(Review.where(user_id: 1))
```

発行される SQL はこんな感じでちゃんといけてます。

```sql
 SELECT
    "reviews".*
 FROM "reviews"
 WHERE ("reviews"."id" = $1 OR "reviews"."user_id" = $2) 
```

# 複数条件の場合
複数条件で OR したい場合でも大丈夫です。

```rb
Review.where(id: 1, user_id: 1).or(Review.where(user_id: 3, star: 4))
```

```sql
 SELECT
    "reviews".*
 FROM "reviews"
 WHERE (
       "reviews"."id" = $1 AND "reviews"."user_id" = $2
    OR "reviews"."user_id" = $3 AND "reviews"."star" = $4
) 
```

ちなみに、複数条件でも条件が一部重複している場合はこんな感じの SQL が発行されます。
今回の場合だと、 id = 1 の条件が重複しています。
```rb
Review.where(id: 1, user_id: 1).or(Review.where(id: 1, star: 4))
```

```sql
 SELECT
    "reviews".*
 FROM "reviews"
 WHERE "reviews"."id" = $1 AND (
       "reviews"."user_id" = $2
    OR "reviews"."star" = $3
) 
```

# さいごに
ActiveRecord 賢いやつ・・・
知れば知るほど便利なのでどんどん網羅していきたいです。

# 参考
https://railsguides.jp/active_record_querying.html#or%E6%9D%A1%E4%BB%B6