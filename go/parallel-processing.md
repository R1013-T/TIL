# Go 言語 並行処理

## goroutine

- funcの前にgoをつけると、並行処理が可能
- この場合、fmt.Println("goroutine invoked")は実行されない
  - main関数が終了すると、goroutineも終了する
  - goroutineの起動中にmain関数が終了してしまっている

```go
go func() {
    defer wg.Done()
    fmt.Println("goroutine invoked")
}()
fmt.Printf("num of working goroutines: %d\n", runtime.NumGoroutine())
fmt.Println("main func finish")
// num of working goroutines: 2
// main func finish
```

### syncGroup

- goroutineの終了を待つには、sync.WaitGroupを使う

```go
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    fmt.Println("goroutine invoked")
}()
wg.Wait()
fmt.Printf("num of working goroutines: %d\n", runtime.NumGoroutine())
fmt.Println("main func finish")
```

### tracer

- goroutineの実行状況を確認するには、tracerを使う
- `go run main.go`を実行すると、`trace.out`が生成される
- `go tool trace trace.out`で、ブラウザで確認できる

```go
func main() {
	f, err := os.Create("trace.out")
	if err != nil {
		log.Fatalln("Error creating trace file:", err)
	}
	defer func() {
		if err := f.Close; err != nil {
			log.Fatalln("Error closing trace file:", err)
		}
	}()
	if err := trace.Start(f); err != nil {
		log.Fatalln("Error starting trace:", err)
	}
	defer trace.Stop()
	ctx, t := trace.NewTask(context.Background(), "main")
	defer t.End()
	fmt.Println("The number of logical CPU Cores:", runtime.NumCPU())

	//task(ctx, "Task1")
	//task(ctx, "Task2")
	//task(ctx, "Task3")

	var wg sync.WaitGroup
	wg.Add(3)
	go cTask(ctx, &wg, "Task1")
	go cTask(ctx, &wg, "Task2")
	go cTask(ctx, &wg, "Task3")
	wg.Wait()

	s := []int{1, 2, 3}
	for _, i := range s {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			fmt.Println(i)
		}(i)
	}
	wg.Wait()

	fmt.Println("main function end")

}
func task(ctx context.Context, name string) {
	// deferにチェーンして関数をつけた場合、最後の関数のみ遅延して実行され、それ以外の関数は即時実行される。
	defer trace.StartRegion(ctx, name).End()
	time.Sleep(time.Second)
	fmt.Println("Hello from", name)
}
func cTask(ctx context.Context, wg *sync.WaitGroup, name string) {
	defer trace.StartRegion(ctx, name).End()
	defer wg.Done()
	time.Sleep(time.Second)
	fmt.Println("Hello from", name)
}
```

## channel

- goroutine間のデータのやり取りには、channelを使う

```go
func main() {
	ch := make(chan int)
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		ch <- 10
		time.Sleep(500 * time.Millisecond)
	}()
	fmt.Println(<-ch) // 10
	wg.Wait()
}
```

### goroutine leak

- goroutineが終了せずに残ってしまうこと

```go
func main() {
	ch1 := make(chan int)
	go func() {
		fmt.Println(<-ch1)
	}()
	fmt.Println("num of working goroutines:", runtime.NumGoroutine())
	// num of working goroutines: 2
}
```

- goleakを使うと、goroutine leakを検知できる

main_test.go
```go
package main

import (
	"go.uber.org/goleak"
	"testing"
)

func TestLeak(t *testing.T) {
	defer goleak.VerifyNone(t)
	main()
}
```
1. `go mod tidy` ... goleakをインストール
2. `go test -v .` ... goroutine leakが検知される

- `ch1 <- 10`を追加することで、goroutine leakを防ぐことができる

```go
func Chanel() {
	ch1 := make(chan int)
	go func() {
		fmt.Println(<-ch1)
	}()
	ch1 <- 10
	fmt.Println("num of working goroutines:", runtime.NumGoroutine())
}
```

### buffered

- channelは、デフォルトでバッファなし
- バッファを設定すると、バッファサイズ分のデータをchannelに格納できる

```go
ch2 := make(chan int, 1)
ch2 <- 2
fmt.Println(<-ch2)
// 2
```

- バッファサイズを超えると、デッドロックになる

```go
ch2 := make(chan int, 1)
  ch2 <- 2
  ch2 <- 3
  fmt.Println(<-ch2)
  // deadlock error
```

### closed

- channelをcloseすると、データの送受信ができなくなる

```go
ch1 := make(chan int)
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    fmt.Println(<-ch1)
}()
ch1 <- 10
close(ch1)

v, ok := <-ch1
fmt.Println(v, ok) // 0 false
wg.Wait()
```

- バッファ付きのchanelをcloseした場合
- バッファに残っているデータは取り出すことができる

```go
ch2 := make(chan int, 2)
ch2 <- 1
ch2 <- 2
close(ch2)

v, ok = <-ch2
fmt.Println(v, ok) // 1 true
v, ok = <-ch2
fmt.Println(v, ok) // 2 true
v, ok = <-ch2
fmt.Println(v, ok) // 0 false
```

### capsel

- カプセル化して、channelを読み取り専用にすることができる

```go
func main() {
  ch3 := generateCountStream()
  for v := range ch3 {
  fmt.Println(v)
}
func generateCountStream() <-chan int {
  ch := make(chan int)
  go func() {
    defer close(ch)
    for i := 0; i < 5; i++ {
      ch <- i
    }
  }()
  return ch
}
```

## select

- 複数のchannelを待ち受けることができる

```go
func main() {
  ch1 := make(chan string)
  ch2 := make(chan string)
  var wg sync.WaitGroup
  wg.Add(2)
  go func() {
    defer wg.Done()
    time.Sleep(500 * time.Millisecond)
    ch1 <- "A"
  }()
  go func() {
    defer wg.Done()
    time.Sleep(800 * time.Millisecond)
    ch2 <- "B"
  }()
  for ch1 != nil || ch2 != nil {
    select {
    case v := <-ch1:
    fmt.Println(v)
    ch1 = nil
    case v := <-ch2:
    fmt.Println(v)
    ch2 = nil
    }
  }
  wg.Wait()
  fmt.Println("finish")
  
    // A
	// B
	// finish
}
```

### timeout context

- contextを使うと、timeoutを設定できる

```go
func Select() {
	ch1 := make(chan string, 1)
	ch2 := make(chan string, 1)
	var wg sync.WaitGroup
	ctx, cancel := context.WithTimeout(context.Background(), 600*time.Millisecond)
	defer cancel()
	wg.Add(2)
	go func() {
		defer wg.Done()
		time.Sleep(500 * time.Millisecond)
		ch1 <- "A"
	}()
	go func() {
		defer wg.Done()
		time.Sleep(800 * time.Millisecond)
		ch2 <- "B"
	}()
loop:
	for ch1 != nil || ch2 != nil {
		select {
		case <-ctx.Done():
			fmt.Println("timeout")
			break loop
		case v := <-ch1:
			fmt.Println(v)
			ch1 = nil
		case v := <-ch2:
			fmt.Println(v)
			ch2 = nil
		}
	}
	wg.Wait()
	fmt.Println("finish")
}

// A
// timeout
// finish
```

### default case

- default caseを使うと、channelが受信できない場合に、default caseが実行される

```go
var wg sync.WaitGroup
	ch := make(chan string, 3)
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < bufSize; i++ {
			time.Sleep(1000 * time.Millisecond)
			ch <- "hello"
		}
	}()
	for i := 0; i < 3; i++ {
		select {
		case m := <-ch:
			fmt.Println(m)
		default:
			fmt.Println("no msg arrived")
		}
		time.Sleep(1500 * time.Millisecond)
	}
	
// no msg arrived
// hello
// hello
```

### receive continuous data

- channelをcloseするまで、データを受信し続ける

```go
const bufSize = 5

func Select() {
	ch1 := make(chan int, bufSize)
	ch2 := make(chan int, bufSize)
	var wg sync.WaitGroup
	ctx, cancel := context.WithTimeout(context.Background(), 3000*time.Millisecond)
	defer cancel()
	wg.Add(3)
	go countProducer(&wg, ch1, bufSize, 50)
	go countProducer(&wg, ch2, bufSize, 500)
	go countConsumer(ctx, &wg, ch1, ch2)
	wg.Wait()
	fmt.Println("finish")
}

func countProducer(wg *sync.WaitGroup, ch chan<- int, size int, sleep int) {
	defer wg.Done()
	defer close(ch)
	for i := 0; i < size; i++ {
		time.Sleep(time.Duration(sleep) * time.Millisecond)
		ch <- i
	}
}
func countConsumer(ctx context.Context, wg *sync.WaitGroup, ch1 <-chan int, ch2 <-chan int) {
	defer wg.Done()
loop:
	for ch1 != nil || ch2 != nil {
		select {
		case <-ctx.Done():
			fmt.Println(ctx.Err())
			break loop
		case v, ok := <-ch1:
			if !ok {
				ch1 = nil
				break
			}
			fmt.Printf("ch1 %v\n", v)
		case v, ok := <-ch2:
			if !ok {
				ch2 = nil
				break
			}
			fmt.Printf("ch2 %v\n", v)
		}
	}
}
```

## mutex + atomic

- 変数i に対して、同時にアクセスしてしまう場合はある
- 並行処理で、同時に変数にアクセスすると、データが壊れる場合がある

```go
func MutexAtomic() {
	var wg sync.WaitGroup
	var i int
	wg.Add(2)
	go func() {
		defer wg.Done()
		i++
	}()
	go func() {
		defer wg.Done()
		i++
	}()
	wg.Wait()
	println(i)
}
```

- `go run -race main.go` を実行すると、データ競合が発生していることがわかる

### mutex

- mutexを使うと、同時に変数にアクセスすることを防ぐことができる

```go
var wg sync.WaitGroup
var mu sync.Mutex
var i int
wg.Add(2)
go func() {
    defer wg.Done()
    mu.Lock()
    defer mu.Unlock()
    i++
}()
go func() {
    defer wg.Done()
    mu.Lock()
    defer mu.Unlock()
    i++
}()
wg.Wait()
println(i)
```

### rwmutex

- 読み込みは複数同時にできるが、書き込みは一つずつしかできない

```go
func MutexAtomic() {
	var wg sync.WaitGroup
	var rwMu sync.RWMutex
	var c int

	wg.Add(4)
	go write(&rwMu, &wg, &c)
	go read(&rwMu, &wg, &c)
	go read(&rwMu, &wg, &c)
	go read(&rwMu, &wg, &c)

	wg.Wait()
	fmt.Println("main exit")
}
func read(mu *sync.RWMutex, wg *sync.WaitGroup, c *int) {
	defer wg.Done()
	time.Sleep(10 * time.Second)
	mu.RLock()
	defer mu.RUnlock()
	fmt.Println("read lock")
	fmt.Println(*c)
	time.Sleep(1 * time.Second)
	fmt.Println("read unlock")
}
func write(mu *sync.RWMutex, wg *sync.WaitGroup, c *int) {
	defer wg.Done()
	mu.Lock()
	defer mu.Unlock()
	fmt.Println("write lock")
	*c++
	time.Sleep(1 * time.Second)
	fmt.Println("write unlock")
}

// write lock
// write unlock
// read lock
// 1
// read lock
// 1
// read lock
// 1
// read unlock
// read unlock
// read unlock
// main exit
```

### atomic

- atomicを使うと、データ競合を防ぐことができる

```go
var wg sync.WaitGroup
	var c int64

	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 10; j++ {
				atomic.AddInt64(&c, 1)
			}
		}()
	}
wg.Wait()
fmt.Println(c)
fmt.Println("main exit")
	
// 50
// main exit
```