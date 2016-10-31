# accepts_nested_atributes_forの子の値をモデルにセット

```
def update
  @parent.attributes = strong_params
end

def strong_params
  params.require(:parent).permit(child_attributes: [:id])
end
```
