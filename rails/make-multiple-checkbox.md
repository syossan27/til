# 複数チェックボックスの作り方

`collection_check_boxes` を使うとサクッと出来る。

- Shop
- ShopItem
- Item

の３つを想定して説明。
中間テーブルを持つ多対多構造。
Itemは商品のマスターテーブル

```
<%= f.collection_check_boxes :item_ids, Item.all, :id, :name %>
```

Strong Parametorの指定は

```
item_ids: []
```

といった形で行うこと。
`:item_ids` のみだと空の値しか入らない。
