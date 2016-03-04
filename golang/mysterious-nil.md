# nilの謎

## 型が違うnilは等価ではない

```
func main() {
  var x *int32 = nil // int32型のnil
  var y *int64 = nil // int64型のnil

  equals(x, y)
  // 結果 false
  return
}

func equals(x, y interface{}) {
  println(x == y)
}
```

## nilと指定した時も同じく等価ではない

```
func main() {
  var x *int32 = nil

  isnil(x)
  return
}

func isnil(x interface{}) {
  println(x == nil)
  // 結果 false
}

// nilであったとしても特別な型が割り振られるわけではなく、
// 文字列や数字と同じように指定した型情報が割り振られる。
```
