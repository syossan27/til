# lock-freeを考える

発端：https://stackimpact.com/docs/go-performance-tuning/#go-performance-patterns  
`Favor lock-free algorithms`

ふむ🤔

[https://www.slideshare.net/kumagi/lock-free-safe]

ほう？🤔

MutexやSemaphoreで排他制御を行なうのではなく、CASなどを使ったlock-freeを実装した方が良いらしい。

## CAS
値の書き換えが想定通りのものなら、実際に書き換えを行なう。  
書き換えが出来るまで、何度でも繰り返す。  
→ 但し、問題もある（ex. ABA問題

上記のCASは有限のステップ数で処理が完了しないため、lock-freeではあるがwait-freeではない。

## Goの場合はどうやるの？
Goの場合、atomicパッケージにCAS実装がある。  
[https://golang.org/pkg/sync/atomic/#CompareAndSwapInt32]

```
package main

import (
  "sync"
  "fmt"
)

var (
  balance int32 = 100
  wg = sync.WaitGroup{}
)

func main() {
  // expect: balance = 55
  var i int32
  for i = 0; i < 10; i++ {
    wg.Add(1)
    go withdraw(i)
  }
  wg.Wait()
  fmt.Println(balance)
}

func withdraw(amount int32) {
  balance = balance - amount
  wg.Done()
}
```

しかし、この方法だと当たり前だがgoroutineの処理タイミングによって全く違った値が算出される。  
そこで、以下のようにCASを用いる。

```

package main

import (
  "sync"
  "fmt"
  "sync/atomic"
)

var (
  balance int32 = 100
  wg = sync.WaitGroup{}
)

func main() {
  // expect: balance = 55
  var i int32
  for i = 0; i < 10; i++ {
    wg.Add(1)
    go withdraw(i)
  }
  wg.Wait()
  fmt.Println(balance)
}

func withdraw(amount int32) {
  for {
    if atomic.CompareAndSwapInt32(&balance, balance, balance - amount) {
      break
    }
  }
  wg.Done()
}
```

すると、想定した55の値が常に算出される。

## パフォーマンスの差異

実際にパフォーマンスに違いは出るのだろうか？  
ベンチマークを取ってみる。

- old: use Mutex
- new: use CAS

```
benchmark         old ns/op     new ns/op     delta
BenchmarkDo-4     4028          3670          -8.89%
BenchmarkDo-4     4000          3683          -7.93%
BenchmarkDo-4     4014          2934          -26.91%
BenchmarkDo-4     3516          3043          -13.45%
BenchmarkDo-4     2759          3660          +32.66%
BenchmarkDo-4     4030          2883          -28.46%
BenchmarkDo-4     4025          3802          -5.54%
BenchmarkDo-4     4027          3358          -16.61%
BenchmarkDo-4     4046          3602          -10.97%
BenchmarkDo-4     3749          3363          -10.30%
```

全体的に早くはなっているが、たまに致命的に遅くなる。

さて、実行するgoroutineの数を増やすとどうなるのだろうか？

- 1000

```
benchmark         old ns/op     new ns/op     delta
BenchmarkDo-4     271236        266445        -1.77%
BenchmarkDo-4     275415        267188        -2.99%
BenchmarkDo-4     269019        269459        +0.16%
BenchmarkDo-4     272291        271073        -0.45%
BenchmarkDo-4     273410        269671        -1.37%
```

- 10000

```
benchmark         old ns/op     new ns/op     delta
BenchmarkDo-4     2782914       2831663       +1.75%
BenchmarkDo-4     2712926       2936499       +8.24%
BenchmarkDo-4     2748464       2891509       +5.20%
BenchmarkDo-4     2810436       2844462       +1.21%
BenchmarkDo-4     2738268       2875502       +5.01%
```

む？なんか段々とlockに負けてるぞ🤔

## Mutexの中身は・・・

結局、CASを使っていたりする。

```
// Fast path: grab unlocked mutex.
if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
  if race.Enabled {
    race.Acquire(unsafe.Pointer(m))
  }
  return
}
```

## 結局どうすりゃいいのか？

多大なコア数を持つCPUでないと、CASなどのlock-freeの効果は薄そう。  
以下のリンク先にも書いてあるが、実際に80coreで動かした場合にMutexがボトルネックになったようである。

https://texlution.com/post/golang-lock-free-values-with-atomic-value/
