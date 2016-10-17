# fields_forの引数を参照する

```
<%= f.fields_for :hoge do |hoge_field| %>
  <%= hoge_field.object.title %>
<% end %>
```

これでhogeモデルのtitle属性を参照できる。
