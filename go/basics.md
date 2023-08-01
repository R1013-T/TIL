# Go 言語 基礎

<!-- TOC -->
* [Go 言語 基礎](#go-言語-基礎)
  * [module・package](#modulepackage)
  * [variable](#variable)
    * [データ型を出力](#データ型を出力)
    * [複数の変数を宣言](#複数の変数を宣言)
    * [型変換](#型変換)
    * [定数](#定数)
  * [pointer](#pointer)
  * [shadowing](#shadowing)
  * [配列](#配列)
    * [スライス](#スライス)
    * [map](#map)
  * [struct(構造体)](#struct構造体)
  * [receiver](#receiver)
  * [function](#function)
    * [defer](#defer)
    * [無名関数](#無名関数)
<!-- TOC -->

## module・package

- module という単位でパッケージを管理する
- module は複数のパッケージを含む
- module の直下に go.mod, go.sum を必ず置く
- package はフォルダで管理する
- 変数と関数を共有できる
    - 同じ package 内での共有は小文字で始める
    - 外部への共有は大文字で始める

## variable

**var 変数名 型 = 値**

- 明示的な代入を省略できる
- 型を省略できる

```go
var message string = "hello world"

//明示的な代入を省略できる
var message string

//型を省略できる
var message = "hello world"
```

**変数名 := 値**

- 明示的な代入が必要（型を推測してくれる）
- 関数内でしか使えない

```go
message := "hello world"

//型をつけることも可能
ui := uint8(1)
```

### データ型を出力

```go
i := int(2)
ui := uint(4)

fmt.Printf("i: %v %T\n", i, i) // i: 2 int
fmt.Printf("i: %[1]v %[1]T ui: %[2]v %[2]T", i, ui) // i: 2 int ui: 4 uint
```

### 複数の変数を宣言
カンマ区切りで複数の変数を宣言できる

```go
pi, title := 3.14, "Go"

fmt.Printf("pi: %v %T\n", pi, pi) // pi: 3.14 float64
fmt.Printf("title: %v %T\n", title, title) // title: Go string

var (
x int
y float64
z uint8
)

```
### 型変換
違う型の変数同士の演算はできないので、型変換が必要
```go
y := 1.23
z := float64(x) + y
fmt.Printf("z: %v %T\n", z, z) // z: 4.23 float64
```

### 定数
**const 定数名 型 = 値**

`iota` ... 連番を生成する

```go

type Os int
const (
Mac Os = iota + 1
Windows
Linux
)

func main() {
fmt.Println(Mac, Windows, Linux)
// 1 2 3
}

```

## pointer
pointer はメモリ上のアドレスを指し示す変数

```go
var ut1 uint16
fmt.Printf("memory address of ut1: %p\n", &ut1)
var ut2 uint16
fmt.Printf("memory address of ut2: %p\n", &ut2)
var p1 *uint16
fmt.Printf("value of p1: %v\n", p1)
p1 = &ut1
fmt.Printf("value of p1: %v\n", p1)
fmt.Printf("size of p1: %d[bytes]\n", unsafe.Sizeof(p1))
fmt.Printf("memory address of p1: %p\n", &p1)
fmt.Printf("valie of ut1(dereference): %v\n", *p1)
*p1 = 1
fmt.Printf("valie of ut1: %v\n", ut1)
fmt.Printf("valie of ut1(dereference): %v\n", *p1)

var pp1 **uint16 = &p1
fmt.Printf("value of pp1: %v\n", pp1)
fmt.Printf("memory address of pp1: %p\n", &pp1)
fmt.Printf("size of pp1: %d[bytes]\n", unsafe.Sizeof(pp1))
fmt.Printf("valie of p1(dereference): %v\n", *pp1)
fmt.Printf("valie of ut1(dereference): %v\n", **pp1)
```

## shadowing
同じスコープ内で同じ名前の変数を宣言することを shadowing という

```go
ok, result := true, "A"
if ok {
result := "B"
println(result)
} else {
result := "C"
println(result)
}
println(result)
```

## 配列
**var 変数名 [要素数]型**
```go
var a1 [3]int
var a2 = [3]int{1, 2, 3}
var a3 = [...]int{1, 2, 3}
var a4 = [...]int{0: 1, 1: 2, 2: 3}
a5 := [...]int{1, 2, 3}

fmt.Printf("%v %v %v %v %v\n", a1, a2, a3, a4, a5)
// [0 0 0] [1 2 3] [1 2 3] [1 2 3] [1 2 3]
```

**`Len()` ... 配列の要素数を返す**
```go
fmt.Println(len(a1)) // 3
```
**`Cap()` ... 配列の容量を返す**
```go
fmt.Println(cap(a1)) // 3
```
**型** ... 配列の要素数も型の一部になる
```go
var a1 [2]int
var a2 = [3]int{1, 2, 3}

fmt.Printf("%T %T\n", a1, a2) // [2]int [3]int
```

### スライス
`var 変数名 []型`... 大きさを指定しない配列

```go
var s1 []int
s2 := []int{}

fmt.Printf("s1: %[1]T %[1]v %v %v\n", s1, len(s1), cap(s1)) // s1: []int [] 0 0
fmt.Printf("s2: %[1]T %[1]v %v %v\n", s2, len(s2), cap(s2)) // s2: []int [] 0 0

fmt.Println(s1 == nil) // true
fmt.Println(s2 == nil) // false
```

**動的に要素数を変更できる**
- `append()` ... 要素を追加
  ```go
  s1 = append(s1, 1, 2, 3)
  fmt.Printf("s1: %[1]T %[1]v %v %v\n", s1, len(s1), cap(s1))
  // s1: []int [1 2 3] 3 4
  ```
  別のスライスを追加する場合は ... をつける   
  ```go
  s3 := []int{4, 5, 6}
  s1 = append(s1, s3...)
  fmt.Printf("s1: %[1]T %[1]v %v %v\n", s1, len(s1), cap(s1))
  // s1: []int [1 2 3 4 5 6] 6 6
  ```
- `make()` ... スライスを作成
  ```go
  s4 := make([]int, 0, 2)
  fmt.Printf("s4: %[1]T %[1]v %v %v\n", s4, len(s4), cap(s4))
  // s4: []int [] 0 2
  
  s4 = append(s4, 1, 2, 3)
  fmt.Printf("s4: %[1]T %[1]v %v %v\n", s4, len(s4), cap(s4))
  // s4: []int [1 2 3] 3 4
  ```
  ```go
  s5 := make([]int, 4, 6)
  fmt.Printf("s5: %[1]T %[1]v %v %v\n", s5, len(s5), cap(s5))
    // s5: []int [0 0 0 0] 4 6
  s6 := s5[1:3]
  s6[1] = 10
  fmt.Printf("s5: %[1]T %[1]v %v %v\n", s5, len(s5), cap(s5))
  fmt.Printf("s6: %[1]T %[1]v %v %v\n", s6, len(s6), cap(s6))
    // s5: []int [0 0 10 0] 4 6
    // s6: []int [0 10] 2 5
  s6 = append(s6, 2)
  fmt.Printf("s5: %[1]T %[1]v %v %v\n", s5, len(s5), cap(s5))
  fmt.Printf("s6: %[1]T %[1]v %v %v\n", s6, len(s6), cap(s6))
      // s5: []int [0 0 10 2] 4 6
	  // s6: []int [0 10 2] 3 5
    ```
- `copy()` ... 要素をコピー
  ```go
  sc6 := make([]int, len(s5[1:3])) // capを指定しない場合は元のcapを引き継ぐ
  fmt.Printf("s5: %[1]T %[1]v %v %v\n", s5, len(s5), cap(s5))
  fmt.Printf("sc6: %[1]T %[1]v %v %v\n", sc6, len(sc6), cap(sc6))
  copy(sc6, s5[1:3])
  fmt.Printf("sc6: %[1]T %[1]v %v %v\n", sc6, len(sc6), cap(sc6))
  sc6[1] = 1
  fmt.Printf("s5: %[1]T %[1]v %v %v\n", s5, len(s5), cap(s5))
  fmt.Printf("sc6: %[1]T %[1]v %v %v\n", sc6, len(sc6), cap(sc6))
  ```

### map
`var 変数名 map[キーの型]値の型` ... キーと値のペアのコレクション

```go
var m1 map[string]int
m2 := map[string]int{}
fmt.Printf("m1: %[1]T %[1]v %v\n", m1, m1 == nil)
fmt.Printf("m2: %[1]T %[1]v %v\n", m2, m2 == nil)
// m1: map[string]int map[] true
// m2: map[string]int map[] false

m2["A"] = 10
m2["B"] = 20
m2["C"] = 30
fmt.Printf("m2: %v %v %v\n", m2, len(m2), m2["A"])
// m2: map[A:10 B:20 C:30] 3 10
```
delete() ... 要素を削除
```go
delete(m2, "A")
fmt.Printf("m2: %v %v %v\n", m2, len(m2), m2["A"])
// m2: map[B:20 C:30] 2 0

v, ok := m2["A"]
fmt.Printf("m2: %v %v %v %v\n", m2, len(m2), v, ok)
// m2: map[B:20 C:30] 2 0 false
v, ok = m2["B"]
fmt.Printf("m2: %v %v %v %v\n", m2, len(m2), v, ok)
// m2: map[B:20 C:30] 2 20 true
```
for文で要素を取り出す(取り出す順番は不定)
```go
for k, v := range m2 {
    fmt.Printf("%v: %v\n", k, v)
}
// B: 20
// C: 30
```

## struct(構造体)
```go
type Task struct {
Title    string
Estimate int
}

func main() {
task1 := Task{
Title:    "Task 1",
Estimate: 3,
}
task1.Title = "Task 1+"
fmt.Printf("%[1]T %+[1]v\n", task1)
// main.Task {Title:"Task 1+", Estimate:3}
}
```
structのコピー ... フィールドの値がコピーされる(別のメモリ領域になる)
```go
var task2 Task = task1
task2.Title = "Task 2"
fmt.Printf("%[1]T %+[1]v\n", task1)
fmt.Printf("%[1]T %+[1]v\n", task2)
// main.Task {Title:"Task 1+", Estimate:3}
// main.Task {Title:"Task 2", Estimate:3}
```

## receiver

```go
...

func main() {
	task1 := Task{
        Title:    "Task 1",
        Estimate: 3,
    }
	
	task1.extendEstimate()
	fmt.Printf("%[1]T %+[1]v\n", task1)
	// main.Task {Title:"Task 1", Estimate:3}
	task1.extendEstimatePointer()
	fmt.Printf("%[1]T %+[1]v\n", task1)
	// main.Task {Title:"Task 1", Estimate:13}
}

func (task Task) extendEstimate() {
	task.Estimate += 10
}
func (taskp *Task) extendEstimatePointer() {
	taskp.Estimate += 10
}
```

## function

### defer
関数の実行を遅延させる
```go
func main() {
    funcDefer()
    // funcDefer
    // defer 2
    // defer 1
}

func funcDefer() {
    defer fmt.Println("defer 1")
    defer fmt.Println("defer 2")
    fmt.Println("funcDefer")
}
```

### 無名関数
```go
i := 1
func(i int) {
fmt.Println(i)
}(i)
// 1

f1 := func(i int) int {
return i + 1
}
fmt.Println(f1(i))
// 2
```