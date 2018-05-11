# Concurrency in Go

## Chapter 1. An Introduction to Concurrency

クラウドコンピューティングによって、エンジニアは簡単にリソースを手に入れれるようになった。  
→ 　しかし異なるマシンを使った同一の課題に対してのモデリングは新たな問題となり、"Web Scale"という新たな複雑さを産んだ。

並行性について2005年に問題提起された記事
http://www.gotw.ca/publications/concurrency-ddj.htm

並行処理は以下の難しさを持つ

### Race conditions

ex.

```
var data int
go func() { 1
    data++
}()
if data == 0 {
    fmt.Printf("the value is %v.\n", data)
}
```

Bad ex.

```
var data int
go func() { data++ }()
time.Sleep(1*time.Second) // This is bad!
if data == 0 {
    fmt.Printf("the value is %v.\n" data)
}
```

### Atomicity

コラム：BlizzardのGliderの話

```
In 2006, the gaming company Blizzard successfully sued MDY Industries for $6,000,000 USD for making a program called “Glider,” which would automatically play their game, World of Warcraft, without user intervention. These types of programs are commonly referred to as “bots” (short for robots).

At the time, World of Warcraft had an anti-cheating program called “Warden,” which would run anytime you played the game. Among other things, Warden would scan the memory of the host machine and run a heuristic to look for programs that appeared to be used for cheating.

Glider successfully avoided this check by taking advantage of the concept of atomic context. Warden considered scanning the memory on the machine as an atomic operation, but Glider utilized hardware interrupts to hide itself before this scanning started! Warden’s scan of memory was atomic within the context of the process, but not within the context of the operating system.
```

原子性は「これ以上分割されない」・「処理が全て完了されるor何も処理しない」を持つ

`i++` は原子性ではない。

- Get value
- Increment value
- Store value

がAtomicではない。

こういったAtomicityの穴を埋めることが大事。

`クリティカルセクション` : 単一のリソースに対して複数処理が同時期に実行されると破綻をきたす部分を指す

Mutexを使ったロックはパフォーマンス低下を招く。  
また正しく動作される可能性はあるが、以下の問題を引き起こす。

- デッドロック： コフマン条件を満たすと発生する
- ライブロック： デッドロックを回避しようと時間差による処理を発生させるが、同時期に処理されることで永遠にロック状態になることを指す（要はお見合い状態）
- 資源飢餓：　作業を実行するために必要なリソースがすべて取得できない状態（デッドロックも資源飢餓状態の一つと言える）

### デッドロック

ex.

```
type value struct {
    mu    sync.Mutex
    value int
}

var wg sync.WaitGroup
printSum := func(v1, v2 *value) {
    defer wg.Done()
    v1.mu.Lock() 1
    defer v1.mu.Unlock() 2

    time.Sleep(2*time.Second) 3
    v2.mu.Lock()
    defer v2.mu.Unlock()

    fmt.Printf("sum=%v\n", v1.value + v2.value)
}

var a, b value
wg.Add(2)
go printSum(&a, &b)
go printSum(&b, &a)
wg.Wait()
```

コフマン条件： 次の４つの条件を指す

#### Mutual Exclusion
A concurrent process holds exclusive rights to a resource at any one time.
リソースは最大１つまでのプロセスにしか確保されないこと

#### Wait For Condition
A concurrent process must simultaneously hold a resource and be waiting for an additional resource.
リソースが確保済みの場合、要求している他のプロセスは待たなければならない

#### No Preemption
A resource held by a concurrent process can only be released by that process, so it fulfills this condition.
リソースは確保したプロセスによってのみ解放される

#### Circular Wait
A concurrent process (P1) must be waiting on a chain of other concurrent processes (P2), which are in turn waiting on it (P1), so it fulfills this final condition too.
リソースを確保しているプロセスAが、他のリソースを確保しているプロセスBのリソースを要求することにより循環待ちが発生する

### ライブロック

ex.

```
cadence := sync.NewCond(&sync.Mutex{})
go func() {
    for range time.Tick(1*time.Millisecond) {
        cadence.Broadcast()
    }
}()

takeStep := func() {
    cadence.L.Lock()
    cadence.Wait()
    cadence.L.Unlock()
}

tryDir := func(dirName string, dir *int32, out *bytes.Buffer) bool { 1
    fmt.Fprintf(out, " %v", dirName)
    atomic.AddInt32(dir, 1) 2
    takeStep() 3
    if atomic.LoadInt32(dir) == 1 {
        fmt.Fprint(out, ". Success!")
        return true
    }
    takeStep()
    atomic.AddInt32(dir, -1) 4
    return false
}

var left, right int32
tryLeft := func(out *bytes.Buffer) bool { return tryDir("left", &left, out) }
tryRight := func(out *bytes.Buffer) bool { return tryDir("right", &right, out) }
```

CPU利用状況を見ても動いてるように見えるので非常に厄介。

### 資源飢餓

ex.

```
var wg sync.WaitGroup
var sharedLock sync.Mutex
const runtime = 1*time.Second

greedyWorker := func() {
    defer wg.Done()

    var count int
    for begin := time.Now(); time.Since(begin) <= runtime; {
        sharedLock.Lock()
        time.Sleep(3*time.Nanosecond)
        sharedLock.Unlock()
        count++
    }

    fmt.Printf("Greedy worker was able to execute %v work loops\n", count)
}

politeWorker := func() {
    defer wg.Done()

    var count int
    for begin := time.Now(); time.Since(begin) <= runtime; {
        sharedLock.Lock()
        time.Sleep(1*time.Nanosecond)
        sharedLock.Unlock()

        sharedLock.Lock()
        time.Sleep(1*time.Nanosecond)
        sharedLock.Unlock()

        sharedLock.Lock()
        time.Sleep(1*time.Nanosecond)
        sharedLock.Unlock()

        count++
    }

    fmt.Printf("Polite worker was able to execute %v work loops.\n", count)
}

wg.Add(2)
go greedyWorker()
go politeWorker()

wg.Wait()
```

資源飢餓はGoプロセス以外がリソースを確保することでも起こるので注意。

### 並行性の安全性

`func CalculatePi(begin, end int64) []uint`

も

`func CalculatePi(begin, end int64) <-chan uint`

とかのシグネチャで返してあげると、内部でgoroutineを生成して実行していると分かりやすい。

このようにパフォーマンスとチーム仕事での分かりやすさを同時にこなさないといけない。

## Chapter 2. Modeling Your Code: Communicating Sequential Processes

### 並行性と並列性の違い

Concurrency is a property of the code.
Parallelism is a property of the running program.

並行性はコードにおける特性・性質の一つであり、  
並列性はプログラム実行における特性・性質の一つである。

参考: https://blog.golang.org/concurrency-is-not-parallelism

Concurrency is about dealing with lots of things at once.
Parallelism is about doing lots of things at once.

並行性は多くのことをこなす。  
並列性は一つのことを多くでこなす。

goroutineの概念はTony Hoare氏の「Communicating sequential processes」の論文の影響を強く受けている。

goroutineはOSのThreadではなく、Goのruntimeが持つThreadより軽量な機能である。

Goのコアチームはsync.Mutexを使った構成より、CSPを使った構成を勧めている節がある。  
ただ、どちらを使ってもいい。  
参考: https://github.com/golang/go/wiki/MutexOrChannel

どちらを使うかの意思決定ツリー

![意思決定ツリー](https://imgur.com/KlgMqZX.jpg)

#### Are you trying to transfer ownership of data?

並行処理結果を別のコードでも使いたいかどうか。  

並行プログラミングで安全に処理する方法の一つとして、あるデータに対して一つの並行コンテキストのみが所有権を持つようにする。  
そういった際にchannelが役立つ。

そうすることの大きな利点の1つは、バッファ付きのチャネルを作成して安価なメモリ内キューを実装して、消費者からプロデューサを切り離すことができることです。  
もう1つは、チャネルを使用することによって、暗黙のうちに並行コードを他の並行コードと組み合わせ可能にしたことです。  
↑ 原文ママだがちょっとよく分からん・・・

#### Are you trying to guard internal state of a struct?

```
type Counter struct {
    mu sync.Mutex
    value int
}
func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}
```

構造体内のプロパティ値をguardするために、Mutexを使わないといけない。

#### Are you trying to coordinate multiple pieces of logic?

複数のロジックが連動するような仕組みの場合には、channelを使うべき。  
オブジェクトグラフ全体にロック機構が広まるのと、channelがあるのとでは実装容易性が違う。

オブジェクトグラフ：オブジェクトとリレーションシップを含めたものを指す

#### Is it a performance-critical section?

パフォーマンスクリティカルセクションはプログラムの再構築が必要なのかも。

これらを指標にすればCSPスタイルの並行処理かメモリアクセス同期(sync.Mutex)のどちらを用いるべきかがはっきりする。  
また、他言語でOS Threadを用いたパターンもあるかもしれないが(例えばスレッドプールなど)、これはGoにとっては当てはまらないことがあるので、真似た実装は避けること。

これらを踏まえ、Goではなるべくchannelを使って実装することが望ましい。

## Chapter 3. Go’s Concurrency Building Blocks

### goroutine

Goのプログラムは全てgoroutineである。  
プログラムの実行時に自動的にmainのgoroutineが生成・実行される。

goroutineはOSのスレッドではなく、グリーンスレッドでもなく、coroutineよりも高位な抽象化である。

グリーンスレッド：オペレーティングシステムではなく仮想マシン (VM) によってスケジュールされるスレッド

M:N scheduler ：N（ユーザー空間）内に複数のM（OSスレッドのランタイムマッピング）を割り当て、G（goroutine）を順次処理していくスケジューラ  
参考: https://morsmachine.dk/go-scheduler

Goの並行性モデルはfork-joinモデル  
![fork-joinモデル](https://imgur.com/q8PIy2r.jpg)

参考: https://en.wikipedia.org/wiki/Fork%E2%80%93join_model

goroutine作成時がforkで、joinにsync.WaitGroupを使ったりする。

#### やらしい問題

```
var wg sync.WaitGroup
for _, salutation := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(salutation)
    }()
}
wg.Wait()
```

これを実行すると

```
good day
good day
good day
```

という出力になる。

goroutineに切り替わるタイミングが

- アンバッファなチャネルへの読み書きが行われる
- システムコールが呼ばれる
- メモリの割り当てが行われる
- time.Sleep()が呼ばれる
- runtime.Gosched()が呼ばれる

であるので、切り替わり時にsalutationの参照が行われて、 `good day` の出力となっている。  
参考：https://qiita.com/niconegoto/items/3952d3c53d00fccc363b

正解はこれ

```
var wg sync.WaitGroup
for _, salutation := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func(salutation string) {
        defer wg.Done()
        fmt.Println(salutation)
    }(salutation)
}
wg.Wait()
```

goroutineの生成時に `salutation` を束縛する。

#### GC

GCはgoroutineを収集しない。  

#### performance

OSスレッドのコンテキストスイッチと、goroutineのchannnel通信とのパフォーマンス差を見る。

- コンテキストスイッチ: 1.467μs
- channel: 0.225μs

92%も高速である。

### sync

#### Cond

排他制御の条件変数（condition variable）にあたる機能。

```
c := sync.NewCond(&sync.Mutex{})
queue := make([]interface{}, 0, 10)

removeFromQueue := func(delay time.Duration) {
    time.Sleep(delay)
    c.L.Lock()
    queue = queue[1:]
    fmt.Println("Removed from queue")
    c.L.Unlock()
    c.Signal()
}

for i := 0; i < 10; i++ {
    c.L.Lock()
    for len(queue) == 2 {
        c.Wait()
    }
    fmt.Println("Adding to queue")
    queue = append(queue, struct{}{})
    go removeFromQueue(1 * time.Second)
    c.L.Unlock()
}
```

このように使えばSemaphore処理として使える。（例はQueueを上限2とした場合のSemaphore処理

Signalを使用した処理の再現はchannelでも出来るが、Broadcastの場合は難しい。  
さらにCondの特徴としてはchannelよりもパフォーマンスに優れる点がある。

Condを使わなければいけない場合がほとんどBroadcastを使うときである。  
Goの標準ライブラリでは主にTLSやHTTP/2のライブラリがサーバーとのハンドシェイクが完了したり、サーバーからレスポンスがきたタイミングでワーカースレッドを起こすのに、sync.Condが使われている。

#### Pool

インスタンスを再利用可能にする機能。

インスタンスを個々に生成することなく、使い回せるのでサイズが大きいインスタンスを扱う際に重宝する。  
また、キャッシングやホットロードのような使い方もできる。

ex.

```
func warmServiceConnCache() *sync.Pool {
    p := &sync.Pool {
        New: connectToService,
    }
    for i := 0; i < 10; i++ {
        p.Put(p.New())
    }
    return p
}

func startNetworkDaemon() *sync.WaitGroup {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        connPool := warmServiceConnCache()

        server, err := net.Listen("tcp", "localhost:8080")
        if err != nil {
            log.Fatalf("cannot listen: %v", err)
        }
        defer server.Close()

        wg.Done()

        for {
            conn, err := server.Accept()
            if err != nil {
                log.Printf("cannot accept connection: %v", err)
                continue
            }
            svcConn := connPool.Get()
            fmt.Fprintln(conn, "")
            connPool.Put(svcConn)
            conn.Close()
        }
    }()
    return &wg
}
```

作成コストがかかるものについては、sync.Poolを使ったキャッシングを行えば段違いのパフォーマンスを得られる。  
都度作成した実装とパフォーマンスを比較したところ、約344倍の違いが見られた。  
注意しなければいけないのは、冪等性のあるインスタンスにしか効果がないということ。  
さもないと、結局都度作成の実装と変わらないものとなる。

sync.Poolで覚えておかないといけない４つのこと

- sync.Poolをインスタンス化するときに、呼び出されるとスレッドセーフである新しいメンバ変数を与えます。
- Getからインスタンスを受け取ったら、受け取ったオブジェクトの状態に関する前提はありません。
- プールから取り出したオブジェクトが終了したら、Putを呼び出すようにしてください。さもなければ、プールは役に立たない。これはdeferで行うべきです。
- プール内のオブジェクトは、均一なオブジェクトでなければいけません。

#### Channel

HoareのCSPに由来する機能。

<-を使って宣言すれば単方向channelが作成出来る。（知らなかった・・・

```
// 送信用
var dataStream <-chan interface{}
dataStream := make(<-chan interface{})

// 受信用
var dataStream chan<- interface{}
dataStream := make(chan<- interface{})
```

主に関数の引数や戻り値の型として使われる。

channelの受信には2値を取る場合がある。

```
value, ok := <-dataStream
```

この `ok` はchannelに別の場所から書き込まれた値か、channelがcloseされたかを表しているbool型です。  
`close(dataStreadm)` とすることで、明確にchannelを閉じることができます。  
閉じたchannelから受信すると `ok:false, value:型ごとのdefault value` が返されます。

また、宣言時にバッファを取ることでQueue処理にすることも可能です。

以下、channelの状態と状態毎に処理した時の結果の表。

| Operation | Channel Statement | Result
|:-----------|:------------|:-------------|
| Read | nil | Block |
| | Open and Not Empty | Value |
| | Open and Empty | Block |
| | Closed | <default value>, false |
| | Write Only | Compilation Error|
| Write | nil | Block |
| | Open and Full | Block |
| | Open and Not Full | Write Value |
| | Closed | panic |
| | Receive Only | Compilation Error |
| Close | nil | panic |
| | Open and Not Empty | Closes Channel; reads succeed until channel is drained, then reads produce default value|
| | Open and Empty | Closes Channel; reads produces default value |
| | Closed | panic |
| | Receive Only | Compilation Error |

このようにchannelはブロッキングによるデッドロックに陥る可能性が高い。  
そのため、channelとgoroutineを1:1とした所有権の原理を導入する。

channelの所有者であるgoroutineは

1. channelのインスタンス化
1. 書き込みの実行、また所有権を別のgoroutineへ移す
1. channelのclose
1. 以上3つをカプセル化し、読み込み専用channelとして公開する

こうすることで以下の効果がある。

- goroutineごとにchannelを初期化しているので、nil channelへの書き込みによるデッドロックを防げる
- goroutineごとにchannelを初期化しているので、nil channelのcloseによるデッドロックを防げる
- channelのcloseタイミングを決めているため、closeされたchannelへの書き込みによるpanicを防げる
- channelのcloseタイミングを決めているため、closeされたchannelのcloseによるpanicを防げる
- channelへの不適切な書き込みを防ぐために、コンパイル時の型チェッカーを利用する

読み込み時には以下のことに憂慮しなくてはいけない。

- channelがいつcloseされるのかを知る
- 何らかの理由でのブロッキングを確実にハンドリングする

前者は2値の返り値の `ok` を調べれば良い。  
後者については

```
The second point is much harder to define because it depends on your algorithm: you may want to time out, you may want to stop reading when someone tells you to, or you may just be content to block for the lifetime of the process.
The important thing is that as a consumer you should handle the fact that reads can and will block.
```

って書いてあるがよく分からん・・・

ex. 

```
chanOwner := func() <-chan int {
    resultStream := make(chan int, 5)
    go func() {
        defer close(resultStream)
        for i := 0; i <= 5; i++ {
            resultStream <- i
        }
    }()
    return resultStream
}

resultStream := chanOwner()
for result := range resultStream {
    fmt.Printf("Received: %d\n", result)
}
fmt.Println("Done receiving!")
```

このコードからカプセル化されたchanOwner内でcloseが複数回発生する可能性は排除されていることが分かる。  
このようにchannelの所有権の法則に従うことで、安全性の高いプログラムになる。  
channelの所有権が及ぶ範囲はなるべく狭くすること。

## Chapter 4. Concurrency Patterns in Go

### Confinement（閉じ込め）

並行処理を書く際に安全に書くための２つの選択肢がある。

- メモリを共有するための同期プリミティブ（ex. sync.Mutex）
- 通信の同期化（ex.channel）

他にもいくつか暗黙的な選択肢がある。

- immutable data
- confinementによってデータをプロテクト

理想的にはimmutable dataを使うのが良い。  
→ データ作成にはデータコピーしかないため、クリティカルセクションが狭まる

confinementも認知負荷を軽くする効果がある。  
confinementの種類には２種類あり、

- ad hoc
- lexical

がある。

#### ad hoc confinement

「これはこうしようね」という規則によって縛るやり方。

ex. 

```
data := make([]int, 4)

loopData := func(handleData chan<- int) {
    defer close(handleData)
    for i := range data {
        handleData <- data[i]
    }
}

handleData := make(chan int)
go loopData(handleData)

for num := range handleData {
    fmt.Println(num)
}
```

この状態ではdataがloopData、handleDataの両方のループで利用できてしまう。  
ただし、今回の場合は「loopDataのみで使うようにしようね」という縛りによって目的を達成しようとしている。

#### lexical confinement

レキシカルスコープを利用して、アクセスを絞るやり方。

ex.

```
chanOwner := func() <-chan int {
    results := make(chan int, 5)
    go func() {
        defer close(results)
        for i := 0; i <= 5; i++ {
            results <- i
        }
    }()
    return results
}

consumer := func(results <-chan int) {
    for result := range results {
        fmt.Printf("Received: %d\n", result)
    }
    fmt.Println("Done receiving!")
}

results := chanOwner()
consumer(results)
```

この例だとchanOwner内にあるdata部に対し、他から干渉することは不可能なため非常に安全な並行処理である。

これに対して、安全ではない並行処理の例を見てみましょう。

ex.

```
printData := func(wg *sync.WaitGroup, data []byte) {
    defer wg.Done()

    var buff bytes.Buffer
    for _, b := range data {
        fmt.Fprintf(&buff, "%c", b)
    }
    fmt.Println(buff.String())
}

var wg sync.WaitGroup
wg.Add(2)
data := []byte("golang")
go printData(&wg, data[:3])
go printData(&wg, data[3:])

wg.Wait()
```

安全な並行処理を書くことによって、データがいつどこで変更されるのかという関心事から解放され、クリティカルセクションもないためパフォーマンスも向上する。

しかし、時には同期プリミティブを使わなければならない場合もあります。

### for-select

既知なので割愛

### Preventing Goroutine Leaks 

goroutineはGCで解放されないパティーンがあるため、goroutineを綺麗に保たなければならない。

goroutineが終了する３つの条件

- 動作の完了
- recover不能なエラーにより、作業が続行出来ない場合
- 作業の終了を伝達され場合

上２つの条件では解放される。  
しかし、キャンセルの場合はgoroutineがcleanupされることを検討しなくてはならない。

ex. goroutine leak

```
doWork := func(strings <-chan string) <-chan interface{} {
    completed := make(chan interface{})
    go func() {
        defer fmt.Println("doWork exited.")
        defer close(completed)
        for s := range strings {
            // Do something interesting
            fmt.Println(s)
        }
    }()
    return completed
}

doWork(nil)
// Perhaps more work is done here
fmt.Println("Done.")
```

この例では、doWorkにnil channelが渡されることにより、文字列の出力が始まらずメモリリークを起こします。  
幸いにもこの例はライフサイクルが短いため、特に問題にはならないが、通常は長期間動作させるためその場合問題になる。

さらに親goroutineが子goroutineの終了に関して、何の権限も持たないのが問題である。  
そのため、親goroutineが子goroutineにキャンセルを通知できるようにする。

ex.

```
doWork := func(
  done <-chan interface{},
  strings <-chan string,
) <-chan interface{} {
    terminated := make(chan interface{})
    go func() {
        defer fmt.Println("doWork exited.")
        defer close(terminated)
        for {
            select {
            case s := <-strings:
                // Do something interesting
                fmt.Println(s)
            case <-done:
                return
            }
        }
    }()
    return terminated
}

done := make(chan interface{})
terminated := doWork(done, nil)

go func() {
    // Cancel the operation after 1 second.
    time.Sleep(1 * time.Second)
    fmt.Println("Canceling doWork goroutine...")
    close(done)
}()

<-terminated
fmt.Println("Done.")
```

こうすることで、子goroutineがどのような状況になろうとも親goroutineから処理をキャンセルすることができる。

また、リークする状況としては以下のように子goroutine側が永遠に終わらない状況も考えられる。

ex.

```
newRandStream := func() <-chan int {
    randStream := make(chan int)
    go func() {
        defer fmt.Println("newRandStream closure exited.") 1
        defer close(randStream)
        for {
            randStream <- rand.Int()
        }
    }()

    return randStream
}

randStream := newRandStream()
fmt.Println("3 random ints:")
for i := 1; i <= 3; i++ {
    fmt.Printf("%d: %d\n", i, <-randStream)
}
```

この状況も先ほどの解決法と同じく、キャンセル処理を実装して解決できる。

ex.

```
newRandStream := func(done <-chan interface{}) <-chan int {
    randStream := make(chan int)
    go func() {
        defer fmt.Println("newRandStream closure exited.")
        defer close(randStream)
        for {
            select {
            case randStream <- rand.Int():
            case <-done:
                return
            }
        }
    }()

    return randStream
}

done := make(chan interface{})
randStream := newRandStream(done)
fmt.Println("3 random ints:")
for i := 1; i <= 3; i++ {
    fmt.Printf("%d: %d\n", i, <-randStream)
}
close(done)

// Simulate ongoing work
time.Sleep(1 * time.Second)
```

このように親goroutineから子goroutineに対してキャンセルを通知することで、メモリリークは防げるが逆に親goroutineではキャンセル通知をするという責任が発生するので気を付けなれければいけない。

### The or-channel 

複数チャンネル内でどれかのチャンネルが終了したら、すべてのチャンネルを終了するという機構。

ex.

```
var or func(channels ...<-chan interface{}) <-chan interface{}
or = func(channels ...<-chan interface{}) <-chan interface{} {
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }

    orDone := make(chan interface{})
    go func() {
        defer close(orDone)

        switch len(channels) {
        case 2:
            select {
            case <-channels[0]:
            case <-channels[1]:
            }
        default:
            select {
            case <-channels[0]:
            case <-channels[1]:
            case <-channels[2]:
            case <-or(append(channels[3:], orDone)...):
            }
        }
    }()
    return orDone
}
```

以下のように、いずれかのチャンネルがタイムアップした場合にすべての処理を打ち切るということが出来る。

ex.

```
sig := func(after time.Duration) <-chan interface{}{ 1
    c := make(chan interface{})
    go func() {
        defer close(c)
        time.Sleep(after)
    }()
    return c
}

start := time.Now() 2
<-or(
    sig(2*time.Hour),
    sig(5*time.Minute),
    sig(1*time.Second),
    sig(1*time.Hour),
    sig(1*time.Minute),
)
fmt.Printf("done after %v", time.Since(start)) 3
```

### Error Handling

並行処理でのエラーハンドリングで考えなければいけないことは「誰がエラーをハンドリングするか？」である。

ex.

```
checkStatus := func(
  done <-chan interface{},
  urls ...string,
) <-chan *http.Response {
    responses := make(chan *http.Response)
    go func() {
        defer close(responses)
        for _, url := range urls {
            resp, err := http.Get(url)
            if err != nil {
                fmt.Println(err) 1
                continue
            }
            select {
            case <-done:
                return
            case responses <- resp:
            }
        }
    }()
    return responses
}

done := make(chan interface{})
defer close(done)

urls := []string{"https://www.google.com", "https://badhost"}
for response := range checkStatus(done, urls...) {
    fmt.Printf("Response: %v\n", response.Status)
}
```

正しい方法としては、この並行プロセスに関する完全な情報をどこかで持っておき、そこでエラーを保持するというやり方。

ex1.

```
type Result struct {
    Error error
    Response *http.Response
}
checkStatus := func(done <-chan interface{}, urls ...string) <-chan Result {
    results := make(chan Result)
    go func() {
        defer close(results)

        for _, url := range urls {
            var result Result
            resp, err := http.Get(url)
            result = Result{Error: err, Response: resp}
            select {
            case <-done:
                return
            case results <- result:
            }
        }
    }()
    return results
}
```

ex2.

```
done := make(chan interface{})
defer close(done)

urls := []string{"https://www.google.com", "https://badhost"}
for result := range checkStatus(done, urls...) {
    if result.Error != nil { 5
        fmt.Printf("error: %v", result.Error)
        continue
    }
    fmt.Printf("Response: %v\n", result.Response.Status)
}
```

ここで重要なのは子goroutineの情報を親goroutineがコントロールできる、という点です。  
エラーハンドリングについては子goroutineから上手く分離できた状態になります。

こうすることで、以下のようなハンドリングができます。

ex.

```

done := make(chan interface{})
defer close(done)

errCount := 0
urls := []string{"a", "https://www.google.com", "b", "c", "d"}
for result := range checkStatus(done, urls...) {
    if result.Error != nil {
        fmt.Printf("error: %v\n", result.Error)
        errCount++
        if errCount >= 3 {
            fmt.Println("Too many errors, breaking!")
            break
        }
        continue
    }
    fmt.Printf("Response: %v\n", result.Response.Status)
}
```

これで３つ以上のエラーが帰ってきた場合には、処理を打ち切るということができます。

### Pipelines

`あんま意味分からんのでもっかい読む`

抽象化のためにパイプラインという仕組みを用いる。  
ステージを用いるメリットとして、各ステージでの問題の分離が出来るなど多数のメリットがある。

ex.

```
multiply := func(values []int, multiplier int) []int {
    multipliedValues := make([]int, len(values))
    for i, v := range values {
        multipliedValues[i] = v * multiplier
    }
    return multipliedValues
}
```

これは整数のスライスをループさせながら乗算する単純なステージです。

ex.

```
add := func(values []int, additive int) []int {
    addedValues := make([]int, len(values))
    for i, v := range values {
        addedValues[i] = v + additive
    }
    return addedValues
}
```

これは整数のスライスをループさせながら和算する単純なステージです。  
さて、これらを組み合わせてみましょう。

ex.

```
ints := []int{1, 2, 3, 4}
for _, v := range add(multiply(ints, 2), 1) {
    fmt.Println(v)
}
```

このように、高次関数やモナドのような処理を書くことが出来る。

それではこれらをchannelを使って書き直してみる。

ex.

```
generator := func(done <-chan interface{}, integers ...int) <-chan int {
    intStream := make(chan int)
    go func() {
        defer close(intStream)
        for _, i := range integers {
            select {
            case <-done:
                return
            case intStream <- i:
            }
        }
    }()
    return intStream
}

multiply := func(
  done <-chan interface{},
  intStream <-chan int,
  multiplier int,
) <-chan int {
    multipliedStream := make(chan int)
    go func() {
        defer close(multipliedStream)
        for i := range intStream {
            select {
            case <-done:
                return
            case multipliedStream <- i*multiplier:
            }
        }
    }()
    return multipliedStream
}

add := func(
  done <-chan interface{},
  intStream <-chan int,
  additive int,
) <-chan int {
    addedStream := make(chan int)
    go func() {
        defer close(addedStream)
        for i := range intStream {
            select {
            case <-done:
                return
            case addedStream <- i+additive:
            }
        }
    }()
    return addedStream
}

done := make(chan interface{})
defer close(done)

intStream := generator(done, 1, 2, 3, 4)
pipeline := multiply(done, add(done, multiply(done, intStream, 2), 1), 2)

for v := range pipeline {
    fmt.Println(v)
}
```

generatorはintスライスを順次分解してchannelに送る、これはデータストリームに変換する処理とも言いかえることが出来る。

![挙動表](https://imgur.com/XTfFKFN.jpg)


