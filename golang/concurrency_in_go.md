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

![意思決定ツリー]("https://imgur.com/a/gafyH4o")

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
![fork-joinモデル]("https://imgur.com/a/lFBex4g")

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
