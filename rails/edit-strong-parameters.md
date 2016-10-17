# Strong Parametersの中身をいじる

```
def hoge_params
  hoge_params = params.require(:hoge).permit(:foo)
  hoge_params['bar'] = 'bar'
  return hoge_params
end
```

hoge_paramsに入ってるのはハッシュなので、それをいじればよい。
