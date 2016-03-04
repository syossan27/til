# Interfaceについての学び

参考元
http://dev.classmethod.jp/go/golang-6/

## 普通のインターフェイス

```
// Carインターフェイスの実装
type Car interface {
  run(int) string
  stop()
}

// MyCar構造体の実装
type MyCar struct {
  name string
  speed int
}

// メソッドの実装
func (u *MyCar) run(speed int) string {
  u.speed = speed
  return strconv.Itoa(speed) + "kmで走ります"
}
func (u * MyCar) stop() {
  fmt.Println("停止します")
  u.speed = 0
}

// ポインタを返す
myCar := &MyCar(name: "マイカー", speed: 0)

var i Car = myCar
fmt.Println(i.run(50))
i.stop()
```

## 空のインターフェイス

```
// 空インターフェイス定義
var x interface{}
num := 0
str := "hello"

// どんな型でも代入可能
x = num
x = str
```

## 実際の型を判定する

```
// 空のインターフェイス
type Element interface{}

// 適当な値をセット
var element Element = "hello"

if value, ok := element.(int); ok {
  fmt.Println("int value:%d", value)
} else if value, ok := element.(string); ok {
  fmt.Println("string value:%d", value)
} else {
  fmt.Println("other")
}

// case-switchバージョン
switch value := element.(type) {
  case int:
    fmt.Println("int value:%d", value)
  case string:
    fmt.Println("string value:%d", value)
  default:
    fmt.Println("other")
}
```

## nilのインターフェイスは必ずしもnilじゃない

```
type MagicError struct{}

func (MagicError) Error() string {
  return "[Magic]"
}

func Generate() *MagicError {
  return nil
}

func Test() error {
  return Generate()
}

func main() {
  if Test() != nil {
    // Hello, Mer. Pike!が出力される・・・
    // nilインターフェイスはnilじゃない
    fmt.Println("Hello, Mr. Pike!")
  }
}
```
