# 一つのform内でsubmitの処理を分ける場合

例えばform内に保存と下書き保存の２つがあった場合

```View
<%= form_for ~ do |f| %>
  <% f.submit '保存' %>
  <% f.submit '下書き保存', name: draft %>
<% end %>
```

```Controller
def create
  if params[:draft]
    # 下書き保存時の処理
  end
end
```

ってな感じでControllerで処理を振り分ける
