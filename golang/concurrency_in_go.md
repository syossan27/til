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

参考：https://blog.golang.org/pipelines

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

しかし、パイプラインのあるステージが非常に遅くなった場合はパイプライン全体のボトルネックになる。  
これを緩和するためにFan-Out, Fan-Inがある。

#### Fan-Out, Fan-In

何もわからない・・・  
あとでもう一度調べる

#### or-done-channel

```
for val := range myChan {
      // Do something with val
}
```

に対して、for-select Loopのルールを当てはめると

```
loop:
for {
    select {
    case <-done:
        break loop
    case maybeVal, ok := <-myChan:
        if ok == false {
            return // or maybe break from for
        }
        // Do something with val
    }
}
```

になるが、この場合の問題点として `Do something with val` の処理如何によってはネストが更に深くなり混乱を招く可能性が生まれる。  
そのため、goroutineを使って以下のように書き直すのが望ましい。

```
orDone := func(done, c <-chan interface{}) <-chan interface{} {
    valStream := make(chan interface{})
    go func() {
        defer close(valStream)
        for {
            select {
            case <-done:
                return
            case v, ok := <-c:
                if ok == false {
                    return
                }
                select {
                case valStream <- v:
                case <-done:
                }
            }
        }
    }()
    return valStream
}
```

こうすることで、キャンセル機構をカプセル化した関数に任せることが出来る。

```
for val := range orDone(done, myChan) {
    // Do something with val
}
```

ネストが一気に浅くなった。

#### The tee-channel

```
tee := func(
    done <-chan interface{},
    in <-chan interface{},
) (_, _ <-chan interface{}) { <-chan interface{}) {
    out1 := make(chan interface{})
    out2 := make(chan interface{})
    go func() {
        defer close(out1)
        defer close(out2)
        for val := range orDone(done, in) {
            var out1, out2 = out1, out2 1
            for i := 0; i < 2; i++ { 2
                select {
                case <-done:
                case out1<-val:
                    out1 = nil 3
                case out2<-val:
                    out2 = nil 3
                }
            }
        }
    }()
    return out1, out2
}
```

channelを複数に分けて、内容を出し分ける。  
goroutineごとに複数値を使う時に重宝する？

使う時はこう。

```
done := make(chan interface{})
defer close(done)

out1, out2 := tee(done, take(done, repeat(done, 1, 2), 4))

for val1 := range out1 {
    fmt.Printf("out1: %v, out2: %v\n", val1, <-out2)
}
```

#### The bridge-channel 

channelのchannelを使って順次処理するパターン。

```
bridge := func(
    done <-chan interface{},
    chanStream <-chan <-chan interface{},
) <-chan interface{} {
    valStream := make(chan interface{})
    go func() {
        defer close(valStream)
        for {
            var stream <-chan interface{}
            select {
            case maybeStream, ok := <-chanStream:
                if ok == false {
                    return
                }
                stream = maybeStream
            case <-done:
                return
            }
            for val := range orDone(done, stream) {
                select {
                case valStream <- val:
                case <-done:
                }
            }
        }
    }()
    return valStream
}
```

```
genVals := func() <-chan <-chan interface{} {
    chanStream := make(chan (<-chan interface{}))
    go func() {
        defer close(chanStream)
        for i := 0; i < 10; i++ {
            stream := make(chan interface{}, 1)
            stream <- i
            close(stream)
            chanStream <- stream
        }
    }()
    return chanStream
}

for v := range bridge(nil, genVals()) {
    fmt.Printf("%v ", v)
}
```

channelのchannelの他の使い方ってなんかあるのか調べたい。

#### Queuing

早期にQueingを導入するとdeadlockやlivelockなどの問題を隠蔽してしまうので注意。  
Queingのメリットはパフォーマンス上のメリットではない。

以下さっぱり分からん

#### context package

contextパッケージの利点

- コールグラフの分岐をキャンセルするためのAPIを用意している
- コールグラフを介してリクエストスコープのデータを転送するためのデータバッグを提供する

コールグラフ：コンピュータプログラムのサブルーチン同士の呼び出し関係を表現した有向グラフ   
リクエストスコープ：リクエスト内でのみ使用できる情報、もしくはその「範囲」

また、Preventing Goroutine Leaksの項でも述べたようにキャンセルに焦点を合わすと３つの側面が浮かび上がる

- 親goroutineがキャンセルをしたい
- goroutineはぶら下がる子goroutineをキャンセルしたい
- goroutineのブロッキング操作はキャンセル可能なようにプリエンプティブにする必要がある

contextパッケージは、これら3つすべてを管理するのに役立ちます。

> この精神では、コンテキストのインスタンスはプログラムのコールグラフを流れるようになっています。
> オブジェクト指向のパラダイムでは、頻繁に使用されるデータへの参照をメンバ変数として格納するのが一般的ですが、context.Contextのインスタンスではこれを行わないことが重要です。
> context.Contextのインスタンスは外部から見えるかもしれませんが、内部的にはスタックフレームごとに変更されることがあります。
> このため、常にContextのインスタンスを関数に渡すことが重要です。

contextを使う前のコード

```
func main() {
    var wg sync.WaitGroup
    done := make(chan interface{})
    defer close(done)

    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := printGreeting(done); err != nil {
            fmt.Printf("%v", err)
            return
        }
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := printFarewell(done); err != nil {
            fmt.Printf("%v", err)
            return
        }
    }()

    wg.Wait()
}

func printGreeting(done <-chan interface{}) error {
    greeting, err := genGreeting(done)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", greeting)
    return nil
}

func printFarewell(done <-chan interface{}) error {
    farewell, err := genFarewell(done)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", farewell)
    return nil
}

func genGreeting(done <-chan interface{}) (string, error) {
    switch locale, err := locale(done); {
    case err != nil:
        return "", err
    case locale == "EN/US":
        return "hello", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

func genFarewell(done <-chan interface{}) (string, error) {
    switch locale, err := locale(done); {
    case err != nil:
        return "", err
    case locale == "EN/US":
        return "goodbye", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

func locale(done <-chan interface{}) (string, error) {
    select {
    case <-done:
        return "", fmt.Errorf("canceled")
    case <-time.After(1*time.Minute):
    }
    return "EN/US", nil
}
```

使用後

```
func main() {
    var wg sync.WaitGroup
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    wg.Add(1)
    go func() {
        defer wg.Done()

        if err := printGreeting(ctx); err != nil {
            fmt.Printf("cannot print greeting: %v\n", err)
            cancel()
        }
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := printFarewell(ctx); err != nil {
            fmt.Printf("cannot print farewell: %v\n", err)
        }
    }()

    wg.Wait()
}

func printGreeting(ctx context.Context) error {
    greeting, err := genGreeting(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", greeting)
    return nil
}

func printFarewell(ctx context.Context) error {
    farewell, err := genFarewell(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", farewell)
    return nil
}

func genGreeting(ctx context.Context) (string, error) {
    ctx, cancel := context.WithTimeout(ctx, 1*time.Second)
    defer cancel()

    switch locale, err := locale(ctx); {
    case err != nil:
        return "", err
    case locale == "EN/US":
        return "hello", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

func genFarewell(ctx context.Context) (string, error) {
    switch locale, err := locale(ctx); {
    case err != nil:
        return "", err
    case locale == "EN/US":
        return "goodbye", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

func locale(ctx context.Context) (string, error) {
    select {
    case <-ctx.Done():
        return "", ctx.Err()
    case <-time.After(1 * time.Minute):
    }
    return "EN/US", nil
}
```

1秒以上の処理時間でタイムアウト、またgreeting（挨拶）が失敗したらfarewell（お別れ）をキャンセルするよう実装している。   
またdeadlineの設定があるかどうかを調べるには以下のように書き直す。

```
func locale(ctx context.Context) (string, error) {
    if deadline, ok := ctx.Deadline(); ok {
        if deadline.Sub(time.Now().Add(1*time.Minute)) <= 0 {
            return "", context.DeadlineExceeded
        }
    }

    select {
    case <-ctx.Done():
        return "", ctx.Err()
    case <-time.After(1 * time.Minute):
    }
    return "EN/US", nil
}
```

contextパッケージを使うなら、循環参照の可能性を省くためにdata typeに関するパッケージを据えた方がいいと書いてあるっぽい。   
また、 `WithValue` については型安全ではないので、注意が必要。リクエストスコープ内のデータを扱うためと理解する。

> Use context values only for request-scoped data that transits processes and
> API boundaries, not for passing optional parameters to functions.

参考：https://deeeet.com/writing/2017/02/23/go-context-value/

作者は「処理がContextのValueによって振る舞いが変わる場合、オプションパラメータの領域になっている可能性が高い」と述べている。  
context valueはimmutableであるべき。

以下、作者が考えるcontext valueに関する経験則。

```
1) The data should transit process or API boundaries.
If you generate the data in your process’ memory, it’s probably not a good candidate to be request-scoped data unless you also pass it across an API boundary.

2) The data should be immutable.
If it’s not, then by definition what you’re storing did not come from the request.

3) The data should trend toward simple types.
If request-scoped data is meant to transit process and API boundaries, it’s much easier for the other side to pull this data out if it doesn’t also have to import a complex graph of packages.

4) The data should be data, not types with methods.
Operations are logic and belong on the things consuming this data.

5) The data should help decorate operations, not drive them.
If your algorithm behaves differently based on what is or isn’t included in its Context, you have likely crossed over into the territory of optional parameters.
```

また、関数の引数として値を渡すか、context valueとして値を渡し、目に見えない依存を作るかで色々と意見が分かれそうなので事前にチームで話し合うこと。

## Chapter 5. Concurrency at Scale

### Error Propagation

> Many developers make the mistake of thinking of error propagation as secondary, or “other,” to the flow of their system. 
> Careful consideration is given to how data flows through the system, but errors are something that are tolerated and ferried up the stack without much thought, and ultimately dumped in front of the user. 
> Go attempted to correct this bad practice by forcing users to handle errors at every frame in the call stack, but it’s still common to see errors treated as second-class citizens to the system’s control flow. 
> With just a little forethought, and minimal overhead, you can make your error handling an asset to your system, and a delight to your users.

エラーの内容として必要なもの：

- 何が起こったのか？
- いつ、どこで発生したのか？（完全なスタックトレースが必要、但しメッセージとして出す必要性は無い。また分散システムにおいては物理的にどのマシンでエラーになったか？の情報も必要）
- ユーザーフレンドリーなメッセージ
- ユーザーがより多くのエラー情報を取得できる（エラーの完全表示に対応する）

エラー種別としては２種類

- バグ
- known edge case, 既知のエッジケース（ディスク書き込みの失敗、ネットワークコネクションの切断、etc...）

以下のようなCLI Component→Intermediary(媒介) Component→Low Level Componentのようなシステムについて考えてみましょう。

![component](https://imgur.com/ycZRtaR.jpg)

各コンポーネントの境界で以下のようなエラーに関するラッピング構造が必要となります。

```
func PostReport(id string) error {
    result, err := lowlevel.DoWork()
    if err != nil {
        if _, ok := err.(lowlevel.Error); ok {
            err = WrapErr(err, "cannot post report with id %q", id)
        }
        return err
    }
 // ...
}
```

何故かと言うと、ここで各モジュールが指定しているエラータイプに合致しているかという判断をしなくてはいけないためです。  
エラータイプにはエラーとして内部的に持たなければいけない情報が入るはずです。  
またknown edge caseとバグについてもこのラッピング構造で区別することが出来ます。  

ユーザー側のエラーメッセージとして、エラーIDを含み表示することで興味のあるユーザーにエラー内容を調べる余地を残す。

それらを踏まえて、以下のコード。

```
type MyError struct {
    Inner      error
    Message    string
    StackTrace string
    Misc       map[string]interface{}
}

func wrapError(err error, messagef string, msgArgs ...interface{}) MyError {
    return MyError{
        Inner:      err,
        Message:    fmt.Sprintf(messagef, msgArgs...),
        StackTrace: string(debug.Stack()),
        Misc:       make(map[string]interface{}),
    }
}

func (err MyError) Error() string {
    return err.Message
}
```

次はLowLevel Module

```
type LowLevelErr struct {
    error
}

func isGloballyExec(path string) (bool, error) {
    info, err := os.Stat(path)
    if err != nil {
        return false, LowLevelErr{(wrapError(err, err.Error()))}
    }
    return info.Mode().Perm()&0100 == 0100, nil
}
```

さらにIntermediate Module

```
type IntermediateErr struct {
    error
}

func runJob(id string) error {
    const jobBinPath = "/bad/job/binary"
    isExecutable, err := isGloballyExec(jobBinPath)
    if err != nil {
        return err
    } else if isExecutable == false {
        return wrapError(nil, "job binary is not executable")
    }

    return exec.Command(jobBinPath, "--id="+id).Run()
}
```

最後にCLI Module

```
func handleError(key int, err error, message string) {
    log.SetPrefix(fmt.Sprintf("[logID: %v]: ", key))
    log.Printf("%#v", err) 3
    fmt.Printf("[%v] %v", key, message)
}

func main() {
    log.SetOutput(os.Stdout)
    log.SetFlags(log.Ltime|log.LUTC)

    err := runJob("1")
    if err != nil {
        msg := "There was an unexpected issue; please report this as a bug."
        if _, ok := err.(IntermediateErr); ok {
            msg = err.Error()
        }
        handleError(1, err, msg)
    }
}
```

これを実行すると以下のようなログになる。

```
  [logID: 1]: 21:46:07 main.LowLevelErr{error:main.MyError{Inner:
  (*os.PathError)(0xc4200123f0),
  Message:"stat /bad/job/binary: no such file or directory",
  StackTrace:"goroutine 1 [running]:
  runtime/debug.Stack(0xc420012420, 0x2f, 0xc420045d80)
      /home/kate/.guix-profile/src/runtime/debug/stack.go:24 +0x79
  main.wrapError(0x530200, 0xc4200123f0, 0xc420012420, 0x2f, 0x0, 0x0,
  0x0, 0x0, 0x0, 0x0, ...)
      /tmp/babel-79540aE/go-src-7954NTK.go:22 +0x62
  main.isGloballyExec(0x4d1313, 0xf, 0xc420045eb8, 0x487649, 0xc420056050)
      /tmp/babel-79540aE/go-src-7954NTK.go:37 +0xaa
  main.runJob(0x4cfada, 0x1, 0x4d4c35, 0x22)
      /tmp/babel-79540aE/go-src-7954NTK.go:47 +0x48
  main.main()
      /tmp/babel-79540aE/go-src-7954NTK.go:67 +0x63
  ", Misc:map[string]interface {}{}}}
```

標準出力に表示されるメッセージとしては以下のような形に。

```
[1] There was an unexpected issue; please report this as a bug.
```

Intermediate Moduleのエラー内容が抽象的過ぎるため、少し修正。

```
type IntermediateErr struct {
    error
}

func runJob(id string) error {
    const jobBinPath = "/bad/job/binary"
    isExecutable, err := isGloballyExec(jobBinPath)
    if err != nil {
        return IntermediateErr{wrapError(
            err,
            "cannot run job %q: requisite binaries not available",
            id,
        )} 1
    } else if isExecutable == false {
        return wrapError(
            nil,
            "cannot run job %q: requisite binaries are not executable",
            id,
        )
    }

    return exec.Command(jobBinPath, "--id="+id).Run()
}
```

これでログはこんな感じに。

```
  [logID: 1]: 22:11:04 main.IntermediateErr{error:main.MyError
  {Inner:main.LowLevelErr{error:main.MyError{Inner:(*os.PathError)
  (0xc4200123f0), Message:"stat /bad/job/binary: no such file or directory",
  StackTrace:"goroutine 1 [running]:
  runtime/debug.Stack(0xc420012420, 0x2f, 0x0)
      /home/kate/.guix-profile/src/runtime/debug/stack.go:24 +0x79
  main.wrapError(0x530200, 0xc4200123f0, 0xc420012420, 0x2f, 0x0, 0x0,
  0x0, 0x0, 0x0, 0x0, ...)
      /tmp/babel-79540aE/go-src-7954DTN.go:22 +0xbb
  main.isGloballyExec(0x4d1313, 0xf, 0x4daecc, 0x30, 0x4c5800)
      /tmp/babel-79540aE/go-src-7954DTN.go:39 +0xc5
  main.runJob(0x4cfada, 0x1, 0x4d4c19, 0x22)
      /tmp/babel-79540aE/go-src-7954DTN.go:51 +0x4b
  main.main()
      /tmp/babel-79540aE/go-src-7954DTN.go:71 +0x63
  ", Misc:map[string]interface {}{}}}, Message:"cannot run job \"1\":
  requisite binaries not available", StackTrace:"goroutine 1 [running]:
  runtime/debug.Stack(0x4d63f0, 0x33, 0xc420045e40)
      /home/kate/.guix-profile/src/runtime/debug/stack.go:24 +0x79
  main.wrapError(0x530380, 0xc42000a370, 0x4d63f0, 0x33,
  0xc420045e40, 0x1, 0x1, 0x0, 0x0, 0x0, ...)
      /tmp/babel-79540aE/go-src-7954DTN.go:22 +0xbb
  main.runJob(0x4cfada, 0x1, 0x4d4c19, 0x22)
      /tmp/babel-79540aE/go-src-7954DTN.go:53 +0x356
  main.main()
      /tmp/babel-79540aE/go-src-7954DTN.go:71 +0x63
  ", Misc:map[string]interface {}{}}}

```

ユーザーメッセージはこう。

```
[1] cannot run job "1": requisite binaries not available
```

### Timeouts and Cancellation

タイムアウトが必要な理由：

- リトライされる可能性が低い場合
- リクエストを処理するリソース（ディスク容量、メモリ、etc...）が無い場合
- デッドロックの阻止のため（ライブロックになる可能性もあるが、大規模システムの場合タイミングがかち合う可能性は低い

キャンセルが必要な理由：

- 時間の長い処理の場合にはユーザーに進捗を見せないといけない、その時にユーザーの手でキャンセル処理をされる場合があるため
- 親goroutineが子goroutineに対してキャンセルする場合があるため
- 複数プロセスの中で早く処理が終わったものを採用し、他をキャンセルする場合があるため

それではいつでもキャンセルできる機構を考えてみる。

```
var value interface{}
select {
case <-done:
    return
case value = <-valueStream:
}

result := reallyLongCalculation(value)

select {
case <-done:
    return
case resultStream<-result:
}
```

↑のコードには大きな問題がある。   
それはこのgoroutineが実行される時、 `reallyLongCalculation` によって大きくブロッキングされキャンセル処理をその間受け付けないからだ。   
それでは `reallyLongCalculation` をプリエンプティブにしよう。

```
reallyLongCalculation := func(
    done <-chan interface{},
    value interface{},
) interface{} {
    intermediateResult := longCalculation(value)
    select {
    case <-done:
        return nil
    default:
    }

    return longCaluclation(intermediateResult)
}
```

さて、ここで更に問題なのが、 `longCalculation` でも同様の問題が起こるということ。   
そう、本当にプリエンプティブにしなければいけないのは `longCalculation` だったのです。  
というわけで、 `longCalculation` をプリエンプティブにした上で、 `reallyLOngCalculation` は以下のようにしましょう。

```
reallyLongCalculation := func(
    done <-chan interface{},
    value interface{},
) interface{} {
    intermediateResult := longCalculation(done, value)
    return longCaluclation(done, intermediateResult)
}
```

ここで大事なのは、goroutineを正しく小さく分け、atomicな箇所をプリエンプティブにしていくことです。  

さらに、キャンセルされた後の処理としてロールバックしなければいけません。  
以下の例は悪手な例です。

```
result := add(1, 2, 3)
writeTallyToState(result)
result = add(result, 4, 5, 6)
writeTallyToState(result)
result = add(result, 7, 8, 9)
writeTallyToState(result)
```

この場合、３回目の `writeTallyToState` でキャンセルされ、ロールバックが必要な事態になった場合、どうすればいいでしょうか？   
以下の場合ならそれを考える必要性は薄くなります。

```
result := add(1, 2, 3, 4, 5, 6, 7, 8, 9)
writeTallyToState(result)
```

なぜなら、 `writeTallyToState` 内で正しくロールバックするよう組み込めばいいだけなのですから。

また別の問題として考えなければいけないのは、メッセージ重複です。  
以下のようなパイプライン処理の場合、ステージBが重複してメッセージを受け取る可能性があります。

![duplicateMessage](https://imgur.com/ot4ncmY.jpg)

既に、GeneratorがメッセージをステージAに投げた後にキャンセルされた場合、後続処理が止められず重複メッセージになるという問題です。

対策として最も簡単なものが、子goroutineが走り出して結果を受け取るまで親goroutineはキャンセルを受け付けないようにする方法です。  
これはheartbeatの章で詳細に話します。

他のアプローチとしては、

- 重複メッセージを許容し、キャンセルを受け付けた前か後のどちらかの情報を渡す。
- メッセージを送信する前に親goroutineに許可を得るようにする（heartbeatに似ている）

後者については以下のようなイメージ

![likeHeartbeat](https://imgur.com/vEK3zOu.jpg)

しかし、この方法は実際の所heartbeatよりも複雑になるため、heartbeatを使う方が良い。

### Heartbeats

この章では２種類のheartbeatsを説明する。

- 時間間隔ごとのheartbeats
- 作業単位ごとのheartbeats

それではheartbeatsなコードを見てみる。

```
doWork := func(
    done <-chan interface{},
    pulseInterval time.Duration,
) (<-chan interface{}, <-chan time.Time) {
    heartbeat := make(chan interface{})
    results := make(chan time.Time)
    go func() {
        defer close(heartbeat)
        defer close(results)

        pulse := time.Tick(pulseInterval)
        workGen := time.Tick(2*pulseInterval)

        sendPulse := func() {
            select {
            case heartbeat <-struct{}{}:
            default:
            }
        }
        sendResult := func(r time.Time) {
            for {
                select {
                case <-done:
                    return
                case <-pulse:
                    sendPulse()
                case results <- r:
                    return
                }
            }
        }

        for {
            select {
            case <-done:
                return
            case <-pulse:
                sendPulse()
            case r := <-workGen:
                sendResult(r)
            }
        }
    }()
    return heartbeat, results
}
```

結果を送信している間、入力を待つ間にもheartbeatsのパルス送信をする必要があります。

それではこのheartbeatsを実装した `doWork` を使った例を見てみましょう。

```
done := make(chan interface{})
time.AfterFunc(10*time.Second, func() { close(done) })

const timeout = 2*time.Second
heartbeat, results := doWork(done, timeout/2)
for {
    select {
    case _, ok := <-heartbeat:
        if ok == false {
            return
        }
        fmt.Println("pulse")
    case r, ok := <-results:
        if ok == false {
            return
        }
        fmt.Printf("results %v\n", r.Second())
    case <-time.After(timeout):
        return
    }
}
```

1s毎にheartbeats pulseを送り、10sにはtimeoutとなる処理です。  
結果はこうなります。

```
pulse
pulse
results 52
pulse
pulse
results 54
pulse
pulse
results 56
pulse
pulse
results 58
pulse
```

次に、擬似的にpanicを引き起こし、途中で処理が止まる状況を試してみましょう。

```
doWork := func(
    done <-chan interface{},
    pulseInterval time.Duration,
) (<-chan interface{}, <-chan time.Time) {
    heartbeat := make(chan interface{})
    results := make(chan time.Time)
    go func() {
        pulse := time.Tick(pulseInterval)
        workGen := time.Tick(2*pulseInterval)

        sendPulse := func() {
            select {
            case heartbeat <-struct{}{}:
            default:
            }
        }
        sendResult := func(r time.Time) {
            for {
                select {
                case <-pulse:
                    sendPulse()
                case results <- r:
                    return
                }
            }
        }

        for i := 0; i < 2; i++ { // 擬似的にpanicを引き起こす
            select {
            case <-done:
                return
            case <-pulse:
                sendPulse()
            case r := <-workGen:
                sendResult(r)
            }
        }
    }()
    return heartbeat, results
}

done := make(chan interface{})
time.AfterFunc(10*time.Second, func() { close(done) })

const timeout = 2 * time.Second
heartbeat, results := doWork(done, timeout/2)
for {
    select {
    case _, ok := <-heartbeat:
        if ok == false {
            return
        }
        fmt.Println("pulse")
    case r, ok := <-results:
        if ok == false {
            return
        }
        fmt.Printf("results %v\n", r)
    case <-time.After(timeout):
        fmt.Println("worker goroutine is not healthy!")
        return
    }
}
```

結果は以下。

```
pulse
pulse
worker goroutine is not healthy!
```

panicでheartbeats pulseが途絶えてから2s後には異常を感知して終了しいることが分かります。   
これにより、デッドロックを回避できていることが分かります。  
このことについては"Healing Unhealthy Goroutines"の項で深掘りしていきます。

次は、作業単位ごとのheartbeatsです。

```
doWork := func(done <-chan interface{}) (<-chan interface{}, <-chan int) {
    heartbeatStream := make(chan interface{}, 1)
    workStream := make(chan int)
    go func () {
        defer close(heartbeatStream)
        defer close(workStream)

        for i := 0; i < 10; i++ {
            select {
            case heartbeatStream <- struct{}{}:
            default:
            }

            select {
            case <-done:
                return
            case workStream <- rand.Intn(10):
            }
        }
    }()

    return heartbeatStream, workStream
}

done := make(chan interface{})
defer close(done)

heartbeat, results := doWork(done)
for {
    select {
    case _, ok := <-heartbeat:
        if ok {
            fmt.Println("pulse")
        } else {
            return
        }
    case r, ok := <-results:
        if ok {
            fmt.Printf("results %v\n", r)
        } else {
            return
        }
    }
}
```

生成した乱数を送信する毎にheartbeat pulseを送っている。  
結果は以下。

```
pulse
results 1
pulse
results 7
pulse
results 7
pulse
results 9
pulse
results 1
pulse
results 8
pulse
results 5
pulse
results 0
pulse
results 6
pulse
results 0
```

この作業単位でのheartbeatsが役立つのはテストを書く時です。  
以下はselect statementに入る前に遅延したら、という状況を再現したコードです。

```
func DoWork(
    done <-chan interface{},
    nums ...int,
) (<-chan interface{}, <-chan int) {
    heartbeat := make(chan interface{}, 1)
    intStream := make(chan int)
    go func() {
        defer close(heartbeat)
        defer close(intStream)

        time.Sleep(2*time.Second)

        for _, n := range nums {
            select {
            case heartbeat <- struct{}{}:
            default:
            }

            select {
            case <-done:
                return
            case intStream <- n:
            }
        }
    }()

    return heartbeat, intStream
}
```

それではこの関数をテストしてみましょう。   
まずは悪い例です。

```
func TestDoWork_GeneratesAllNumbers(t *testing.T) {
    done := make(chan interface{})
    defer close(done)

    intSlice := []int{0, 1, 2, 3, 5}
    _, results := DoWork(done, intSlice...)

    for i, expected := range intSlice {
        select {
        case r := <-results:
            if r != expected {
                t.Errorf(
                  "index %v: expected %v, but received %v,",
                  i,
                  expected,
                  r,
                )
            }
        case <-time.After(1 * time.Second):
            t.Fatal("test timed out")
        }
    }
}
```

結果は以下です。

```
go test ./bad_concurrent_test.go
--- FAIL: TestDoWork_GeneratesAllNumbers (1.00s)
    bad_concurrent_test.go:46: test timed out
FAIL
FAIL    command-line-arguments  1.002s
```

このようにテストの結果が成功したり、失敗したりブレます。  
nondeterministic（非決定論的）とも言えます。  
Timeoutを増やすことも出来ますが、失敗すると時間がかかりテスト全体の時間が遅くなります。

それではこれを対策していきましょう。

```
func TestDoWork_GeneratesAllNumbers(t *testing.T) {
    done := make(chan interface{})
    defer close(done)

    intSlice := []int{0, 1, 2, 3, 5}
    heartbeat, results := DoWork(done, intSlice...)

    <-heartbeat

    i := 0
    for r := range results {
        if expected := intSlice[i]; r != expected {
            t.Errorf("index %v: expected %v, but received %v,", i, expected, r)
        }
        i++
    }
}
```

結果は以下。

```
ok command-line-arguments 2.002s
```

また、時間間隔ごとのheartbeatsのテストはこのように書きます。

```
func DoWork(
    done <-chan interface{},
    pulseInterval time.Duration,
    nums ...int,
) (<-chan interface{}, <-chan int) {
    heartbeat := make(chan interface{}, 1)
    intStream := make(chan int)
    go func() {
        defer close(heartbeat)
        defer close(intStream)

        time.Sleep(2*time.Second)

        pulse := time.Tick(pulseInterval)
        numLoop:
        for _, n := range nums {
            for {
                select {
                case <-done:
                    return
                case <-pulse:
                    select {
                    case heartbeat <- struct{}{}:
                    default:
                    }
                case intStream <- n:
                    continue numLoop
                }
            }
        }
    }()

    return heartbeat, intStream
}

func TestDoWork_GeneratesAllNumbers(t *testing.T) {
    done := make(chan interface{})
    defer close(done)

    intSlice := []int{0, 1, 2, 3, 5}
    const timeout = 2*time.Second
    heartbeat, results := DoWork(done, timeout/2, intSlice...)

    <-heartbeat

    i := 0
    for {
        select {
        case r, ok := <-results:
            if ok == false {
                return
            } else if expected := intSlice[i]; r != expected {
                t.Errorf(
                    "index %v: expected %v, but received %v,",
                    i,
                    expected,
                    r,
                )
            }
            i++
        case <-heartbeat:
        case <-time.After(timeout):
            t.Fatal("test timed out")
        }
    }
}
```

長時間動作するgoroutine、テストが必要なgoroutineについてはheartbeatsパターンを採用する価値がある。

### Replicated Requests

HTTPリクエストやデータ処理などでいち早くユーザーにレスポンスを返すために、同一処理をする複数goroutineを動かし、一番早くレスポンスを返したgoroutineの結果を採用する。   
欠点としてはリソースを喰うこと。

```
doWork := func(
    done <-chan interface{},
    id int,
    wg *sync.WaitGroup,
    result chan<- int,
) {
    started := time.Now()
    defer wg.Done()

    // Simulate random load
    simulatedLoadTime := time.Duration(1+rand.Intn(5))*time.Second
    select {
    case <-done:
    case <-time.After(simulatedLoadTime):
    }

    select {
    case <-done:
    case result <- id:
    }

    took := time.Since(started)
    // Display how long handlers would have taken
    if took < simulatedLoadTime {
        took = simulatedLoadTime
    }
    fmt.Printf("%v took %v\n", id, took)
}

done := make(chan interface{})
result := make(chan int)

var wg sync.WaitGroup
wg.Add(10)

for i:=0; i < 10; i++ { 1
    go doWork(done, i, &wg, result)
}

firstReturned := <-result 2
close(done) 3
wg.Wait()

fmt.Printf("Received an answer from #%v\n", firstReturned)
```

これはgoroutineがランダムな時間スリープし、結果を返すプログラムです。

```
8 took 1.000211046s
4 took 3s
9 took 2s
1 took 1.000568933s
7 took 2s
3 took 1.000590992s
5 took 5s
0 took 3s
6 took 4s
2 took 2s
Received an answer from #8
```

今回の結果で一番早く返ってきたのは8番のプロセスですね。  
仮に1つだけのプロセスしか回さなかった場合、最悪5sかかる見込みでした。

この方法は全体の処理時間が均一になるような場合にはリソースを消耗するというデメリットしかありません。  
異なるプロセス、マシン、データストアへのパス、または異なるデータストアへのアクセスという、異なる実行時条件を持つ処理には効果的です。  
またその他の効用として、フォールトトレランスとスケーラビリティを持ちます。

### Rate Limiting

悪意のある攻撃に立ち向かうため、レート制限。  
分散システムにおいても、あるユーザーによる大量アクセスによる障害が他のユーザーのリクエストに影響を与える可能性がある。  
またクラウドサービスを使用している場合、悪意ある大量アクセスにより料金がかさんで死、という自体をアプリケーション側で防げる。

さて、レート制限にはトークンバケットと呼ばれる方法で実装する。  
トークンバケットは簡単に言うと、リソースへのアクセスにトークンを必要とし、そのトークンの最大保持数を制限するというもの。  
例えば、トークンバケットの保持数を5とすると、一度にリソースにアクセスできる上限が5ということ。  
そして、トークンが補充される時間をリフレッシュレートとすれば完成である。

トークンバケットはバースト性（大量の同時リクエスト）を許容しているため、特にそこには制限をかけない。  
全ユーザーがトークンバケットの限界までバーストすることは可能性として低いと考えるためである。

```
func main() {
    defer log.Printf("Done.")
    log.SetOutput(os.Stdout)
    log.SetFlags(log.Ltime | log.LUTC)

    apiConnection := Open()
    var wg sync.WaitGroup
    wg.Add(20)

    for i := 0; i < 10; i++ {
        go func() {
            defer wg.Done()
            err := apiConnection.ReadFile(context.Background())
            if err != nil {
                log.Printf("cannot ReadFile: %v", err)
            }
            log.Printf("ReadFile")
        }()
    }

    for i := 0; i < 10; i++ {
        go func() {
            defer wg.Done()
            err := apiConnection.ResolveAddress(context.Background())
            if err != nil {
                log.Printf("cannot ResolveAddress: %v", err)
            }
            log.Printf("ResolveAddress")
        }()
    }

    wg.Wait()
}

func Open() *APIConnection {
    return &APIConnection{}
}

type APIConnection struct {}

func (a *APIConnection) ReadFile(ctx context.Context) error {
    // Pretend we do work here
    return nil
}

func (a *APIConnection) ResolveAddress(ctx context.Context) error {
    // Pretend we do work here
    return nil
}
```

上記のプログラムを例にレート制限を実装していく。  
パッケージには `golang.org/x/time/rate` を使います。

```
func Open() *APIConnection {
    return &APIConnection{
        rateLimiter: rate.NewLimiter(rate.Limit(1), 1),
    }
}

type APIConnection struct {
    rateLimiter *rate.Limiter
}

func (a *APIConnection) ReadFile(ctx context.Context) error {
    if err := a.rateLimiter.Wait(ctx); err != nil {
        return err
    }
    // Pretend we do work here
    return nil
}

func (a *APIConnection) ResolveAddress(ctx context.Context) error {
    if err := a.rateLimiter.Wait(ctx); err != nil {
        return err
    }
    // Pretend we do work here
    return nil
}
```

これで結果が１つずつ順に出てくる。  
今回は1秒に1リクエストを捌くようにしてある。  
またレート制限をmultipleにした実装が以下。

```
type RateLimiter interface {
    Wait(context.Context) error
    Limit() rate.Limit
}

func MultiLimiter(limiters ...RateLimiter) *multiLimiter {
    byLimit := func(i, j int) bool {
        return limiters[i].Limit() < limiters[j].Limit()
    }
    sort.Slice(limiters, byLimit)
    return &multiLimiter{limiters: limiters}
}

type multiLimiter struct {
    limiters []RateLimiter
}

func (l *multiLimiter) Wait(ctx context.Context) error {
    for _, l := range l.limiters {
        if err := l.Wait(ctx); err != nil {
            return err
        }
    }
    return nil
}

func (l *multiLimiter) Limit() rate.Limit {
    return l.limiters[0].Limit()
}
```

こんな風に使用する。

```
func Open() *APIConnection {
    secondLimit := rate.NewLimiter(Per(2, time.Second), 1)
    minuteLimit := rate.NewLimiter(Per(10, time.Minute), 10)
    return &APIConnection{
        rateLimiter: MultiLimiter(secondLimit, minuteLimit),
    }
}

type APIConnection struct {
    rateLimiter RateLimiter
}

func (a *APIConnection) ReadFile(ctx context.Context) error {
    if err := a.rateLimiter.Wait(ctx); err != nil {
        return err
    }
    // Pretend we do work here
    return nil
}

func (a *APIConnection) ResolveAddress(ctx context.Context) error {
    if err := a.rateLimiter.Wait(ctx); err != nil {
        return err
    }
    // Pretend we do work here
    return nil
}
```

このように、細かいレート制限をかけることもできます。  
最後にもう一つ、ディスクとネットワークについてもレート制限をかけてみましょう。

```
func Open() *APIConnection {
    return &APIConnection{
        apiLimit: MultiLimiter(
            rate.NewLimiter(Per(2, time.Second), 2),
            rate.NewLimiter(Per(10, time.Minute), 10),
        ),
        diskLimit: MultiLimiter(
            rate.NewLimiter(rate.Limit(1), 1),
        ),
        networkLimit: MultiLimiter(
            rate.NewLimiter(Per(3, time.Second), 3),
        ),
    }
}

type APIConnection struct {
    networkLimit,
    diskLimit,
    apiLimit RateLimiter
}

func (a *APIConnection) ReadFile(ctx context.Context) error {
    err := MultiLimiter(a.apiLimit, a.diskLimit).Wait(ctx)
    if err != nil {
        return err
    }
    // Pretend we do work here
    return nil
}

func (a *APIConnection) ResolveAddress(ctx context.Context) error {
    err := MultiLimiter(a.apiLimit, a.networkLimit).Wait(ctx)
    if err != nil {
        return err
    }
    // Pretend we do work here
    return nil
}
```

### Healing Unhealthy Goroutines

長時間実行されるgoroutineを"回復"させるメカニズム。  
heartbeatsパターンを使って実装していきます。

以下のコードではgoroutineのhealthyを監視するロジックをスチュワードと呼びます。

```
type startGoroutineFn func(
    done <-chan interface{},
    pulseInterval time.Duration,
) (heartbeat <-chan interface{})

newSteward := func(
    timeout time.Duration,
    startGoroutine startGoroutineFn,
) startGoroutineFn {
    return func(
        done <-chan interface{},
        pulseInterval time.Duration,
    ) (<-chan interface{}) {
        heartbeat := make(chan interface{})
        go func() {
            defer close(heartbeat)

            var wardDone chan interface{}
            var wardHeartbeat <-chan interface{}
            startWard := func() {
                wardDone = make(chan interface{})
                wardHeartbeat = startGoroutine(or(wardDone, done), timeout/2)
            }
            startWard()
            pulse := time.Tick(pulseInterval)

        monitorLoop:
            for {
                timeoutSignal := time.After(timeout)

                for {
                    select {
                    case <-pulse:
                        select {
                        case heartbeat <- struct{}{}:
                        default:
                        }
                    case <-wardHeartbeat:
                        continue monitorLoop
                    case <-timeoutSignal:
                        log.Println("steward: ward unhealthy; restarting")
                        close(wardDone)
                        startWard()
                        continue monitorLoop
                    case <-done:
                        return
                    }
                }
            }
        }()

        return heartbeat
    }
}
```

さて、上記のコードを用いてみましょう。

```
log.SetOutput(os.Stdout)
log.SetFlags(log.Ltime | log.LUTC)

doWork := func(done <-chan interface{}, _ time.Duration) <-chan interface{} {
    log.Println("ward: Hello, I'm irresponsible!")
    go func() {
        <-done 1
        log.Println("ward: I am halting.")
    }()
    return nil
}
doWorkWithSteward := newSteward(4*time.Second, doWork) 2

done := make(chan interface{})
time.AfterFunc(9*time.Second, func() { 3
    log.Println("main: halting steward and ward.")
    close(done)
})

for range doWorkWithSteward(done, 4*time.Second) {} 4
log.Println("Done")
```

このコードでは4s毎にスチュワードが監視し、 `doWork` は常にキャンセルされるようになっています。  
そして、9s後には終了するようにしてあります。  
結果は以下。

```
18:28:07 ward: Hello, I'm irresponsible!
18:28:11 steward: ward unhealthy; restarting
18:28:11 ward: Hello, I'm irresponsible!
18:28:11 ward: I am halting.
18:28:15 steward: ward unhealthy; restarting
18:28:15 ward: Hello, I'm irresponsible!
18:28:15 ward: I am halting.
18:28:16 main: halting steward and ward.
18:28:16 ward: I am halting.
18:28:16 Done
```

さて、毎回 `doWork` を作成するのも大変なので、クロージャーを用いて解決してみます。

```
doWorkFn := func(
    done <-chan interface{},
    intList ...int,
) (startGoroutineFn, <-chan interface{}) { 1
    intChanStream := make(chan (<-chan interface{})) 2
    intStream := bridge(done, intChanStream)
    doWork := func(
        done <-chan interface{},
        pulseInterval time.Duration,
    ) <-chan interface{} { 3
        intStream := make(chan interface{}) 4
        heartbeat := make(chan interface{})
        go func() {
            defer close(intStream)
            select {
            case intChanStream <- intStream: 5
            case <-done:
                return
            }

            pulse := time.Tick(pulseInterval)

            for {
                valueLoop:
                for _, intVal := range intList {
                    if intVal < 0 {
                        log.Printf("negative value: %v\n", intVal) 6
                        return
                    }

                    for {
                        select {
                        case <-pulse:
                            select {
                            case heartbeat <- struct{}{}:
                            default:
                            }
                        case intStream <- intVal:
                            continue valueLoop
                        case <-done:
                            return
                        }
                    }
                }
            }
        }()
        return heartbeat
    }
    return doWork, intStream
}
```

使用する際にはブリッジチャンネルを用います。

```
log.SetFlags(log.Ltime | log.LUTC)
log.SetOutput(os.Stdout)

done := make(chan interface{})
defer close(done)

doWork, intStream := doWorkFn(done, 1, 2, -1, 3, 4, 5) 1
doWorkWithSteward := newSteward(1*time.Millisecond, doWork) 2
doWorkWithSteward(done, 1*time.Hour) 3

for intVal := range take(done, intStream, 6) { 4
    fmt.Printf("Received: %v\n", intVal)
}
```


