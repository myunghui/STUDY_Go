# 9장. 공유변수를 이용한 동시성

### 9.1 데이터 경쟁 (Data Race)

- 두 goroutine이 동일 변수를 동시에 접근하고, 그 중 최소 1개의 접근에서 변수를 갱신할 때 발생

```go
// 1개의 계좌가 개설된 은행 프로그램 (동시성에 안전하지 않음)
package bank
var balance int 
func Deposit(amount int) { balance = balance + amount } 
func Balance() int { return balance }
```
```go
// Alice: 
go func() {
	bank.Deposit(200) // A1
    fmt.Println("=", bank.Balance()) // A2
}() fmt
```
```go
// Bob:
go bank.Deposit(100) // B
```
- 데이터 경쟁를 회피하는 3가지 방법

  1) 변수를 갱신하지 않는다

​	=> goroutine생성전에 모든 변수를 초기화하며, goroutine은 읽기만 한다.

```go
var icons = map[string]image.Image{ 
    "spades.png": loadIcon("spades.png"), 
    "hearts.png": loadIcon("hearts.png"), 
    "diamonds.png": loadIcon("diamonds.png"), 
    "clubs.png": loadIcon("clubs.png"),
}
// Concurrency-safe. 
func Icon(name string) image.Image { return icons[name] }
```
​	2) 여러 Gorutine에서의 접근을 피한다.

  	=> 변수를 가진 Gorutine에게 Channel로 요청한다.
```go
//gopl.io/ch9/bank1

// 동시성에 안전한 1개의 계좌가 개설된 은행 프로그램 
package bank

var deposits = make(chan int) // send amount to deposit 
var balances = make(chan int) // receive balance

func Deposit(amount int) { deposits <- amount  } 
func Balance() int       { return   <- balances}
func teller() {
	var balance int // teller goroutine 에 한정된 balance 
    for {
        select {
            case amount := <- deposits:
                balance += amount
            case balances <- balance:
        }
    }
}
func init() { 
	go teller() // teller goroutine 시작 
}
```
​	3) 여러 Gorutine에서 변수 접근을 허용하지만, 한 번에 하나씩만 접근한다.

  	=> 상호 배제(mutual excusion)

------

### 9.2  상호 배제 : sync.Mutex

- 이진 세마포어(binary semaphore) 지원하는 Mutex
- 변수 접근의 목적\(쓰기, 읽기) 과 무관하게 1번에 1개의 접근만 허용하도록 하는 잠금 알고리즘
- Mutex의 Lock과 UnLock은 반드시 한쌍이어야 한다.
- Lock호출후 반드시 Unlock이  호출되도록 함수를 충분히 원자성 있게 쪼갠다.

```go
// gopl.io/ch9/bank2
var (
	mu sync.Mutex // balance 변수를 보호하는 뮤텍스. 관행상 보호할 변수위에 선언한다.
    balance int
)
func Deposit(amount int) { 
    // Lock과 UnLock은 반드시 한쌍이어야 한다.
    mu.Lock() // 뮤텍스는 임계 영역(critical section)에서 변수값을 유지하는 목적
    // critical section start
    balance = balance + amount // 쓰기
     // critical section End
    mu.Unlock()
}
func Balance() int { 
    mu.Lock() 
    // critical section start
    b:= balance 				// 읽기
    // critical section End
    mu.Unlock()
    return b
}
```
```go
// Mutex 교착으로 영구 대기
func Withdraw(amount int) bool { 
    mu.Lock() 
    defer mu.Unlock() 
    Deposit(-amount) // Mutex는 Re-entry 허용 안함
    if Balance() < 0 { 
        Deposit(amount) 
        return false // 잔액 부족
	}
    return true
}
```

------
### 9.3  읽기/쓰기 뮤텍스 : sync.RWMutex

- 다중 읽기(비배타적), 단일 쓰기(배타적)잠금 구현해야 한다면 `sync.RWMutex`를 사용

```go
var mu sync.RWMutex 
var balance int

func Withdraw(amount int) bool { 
    mu.Lock() 
    defer mu.Unlock() 
    deposit(-amount) 
    if balance < 0 { 
        deposit(amount) 
        return false // 잔액 부족
    }
    return true
}
func Balance() int { 
    mu.RLock() // 읽기전용 Lock 으로 변경
    defer mu.RUnlock()
    return balance
}
func Deposit(amount int) { 
    mu.Lock() 
    defer mu.Unlock() 
    deposit(amount)
}
// 주의) 사전 필수 조건 : 잠금 획득 상태
func deposit(amount int) { balance += amount }
```
------
### 9.4  메모리 동기화(Memory Synchronization)

- 채널통신과 뮤텍스의 역할
프로세서가 누적된 쓰기 작업을 반환하고 저장하게 함으로써, 지금까지 실행된 고루틴 결과를 다른 고루틴에서 동기화하도록 보장한다.
(효율성을 위해 지역 프로세서들이 쓰기한 모든 메모리가 메인메모리로 공유되진 않기 때문)

```go
var x, y int 
// A 고루틴
go func() {
	x = 1 // A1
	fmt.Print("y:", y, " ") // A2
}()
// B 고루틴
go func() {
	y=1 // B1 
}()

/*
채널 또는 뮤텍스를 사용한 명시적인 동기화가 없으면 이벤트의 순서를 보장하지 않는다.
y:0 x:1 
x:0 y:1 
x:1 y:1 
y:1 x:1
x:0 y:0 
y:0 x:0
 */
```

------
### 9.5  게으른 초기화 : sync.Once

- 고비용의 초기화를 꼭 필요한 시점까지 늦추는 방법 제공
- Once는 개념적으로 뮤텍스와 초기화가 완료되었는지를 기록하는 불리언 변수 가지고 있음
- 사용법 : sync.Once(초기화 함수)

```go
/*
  sync.Once 사용한 코드
*/
var loadIconsOnce sync.Once
var icons map[string]image.Image
// 동시성에 안전
func Icon(name string) image.Image { 
    loadIconsOnce.Do(loadIcons) 
    return icons[name]
}
```

```go
/*
  sync.Once 없이 sync.RWmutex 사용한 코드
*/
var mu sync.RWMutex // guards icons 
var icons map[string]image.Image
// 동시성에 안전
func Icon(name string) image.Image { 
    
    mu.RLock() 		// 읽기 잠금 획득
    if icons != nil { 
        icon := icons[name] 
        mu.RUnlock()// 읽기 잠금 해제
        return icon 
	}
    mu.RUnlock()	// 읽기 잠금 해제
    
    mu.Lock()		// 쓰기 잠금 획득
    // 반드시 다시 nil 체크해봐야 함
    if icons == nil {
        loadIcons()
    }
	icon := icons[name]
    mu.Unlock()		// 쓰기 잠금 해제
    return icon
}
func loadIcons() { 
    icons = make(map[string]image.Image) 
    icons["spades.png"] = loadIcon("spades.png") 
    icons["hearts.png"] = loadIcon("hearts.png") 
    icons["diamonds.png"] = loadIcon("diamonds.png") 
    icons["clubs.png"] = loadIcon("clubs.png")
}
```

------
### 9.6  경쟁 상태 검출 (The RaceDetector)

- 동시성에 관한 실수를 검출해주는 동적 분석 도구 제공
- 이벤트 스크림을 관찰해, 한 고루틴에서 최근에 다른 고루틴이 수정한 공유변수를 동기화과정없이 읽거나 쓰는 경우를 감지
- 실제로 수행된 모든 데이터 경쟁을 보고하지만, 경쟁 상태를 예방할 순 없다.
- 사용법 : `go build`, `go run`, `go test` 명령 뒤에 `-race` 플래그 사용

------
### 9.7  동시 넌블로킹 캐시 예제

- 함수의 결과를 캐시하여 한번만 계산하도록 함
- 동시성에 안전하며, 전체 캐시에 대한 단일 잠금 설계로 경합 상태를 방지

// 예제) 캐시하고 싶은 함수
```go

func httpGetBody(url string) (interface{}, error) { 
    resp, err := http.Get(url) 
    if err != nil {
    	return nil, err 
    }
    defer resp.Body.Close()
	return ioutil.ReadAll(resp.Body)
}
```
// 1단계 : 동기화 없이 캐시만 되도록 설계
```go
package memo

// Memo : 함수와 결과 저장 구조 (노출되어야 함으로 대문자로 시작)
type Memo struct {
	f     Func
	cache map[string]result
}
// Func : 캐시할 함수 구조 (노출되어야 함으로 대문자로 시작)
type Func func(key string) (interface{}, error)
// result : 캐시할 함수 결과 및 에러
type result struct {
	value interface{}
	err   error
}

// Memo 포인터 반환
func New(f Func) *Memo {
	return &Memo{f: f, cache: make(map[string]result)}
}

// 함수 처리 결과 요청 : 동시성에 안전하지 않음
func (memo *Memo) Get(key string) (interface{}, error) {
	res, ok := memo.cache[key]
	if !ok {
		res.value, res.err = memo.f(key)
		memo.cache[key] = res
	}
	return res.value, res.err
}
```
```go
// 호출하는 코드 1
m := memo.New(httpGetBody)
for url := range incomingURLs() { 
    start := time.Now() 
    value, err := m.Get(url) 
    if err != nil {
        log.Print(err) 
    }
    fmt.Printf("%s, %s, %d bytes\n", url, time.Since(start), len(value.([]byte)))
}

// 테스트 결과
$gotest -v gopl.io/ch9/memo1 
=== RUN Test 
https://golang.org, 175.026418ms, 7537 bytes   <= 1차
https://godoc.org, 172.686825ms, 6878 bytes 
https://play.golang.org, 115.762377ms, 5767 bytes 
http://gopl.io, 749.887242ms, 2856 bytes

https://golang.org, 721ns, 7537 bytes          <= 2차
https://godoc.org, 152ns, 6878 bytes 
https://play.golang.org, 205ns, 5767 bytes 
http://gopl.io, 326ns, 2856 bytes
--- PASS: Test (1.21s) 
PASS
ptg16091132
ok gopl.io/ch9/memo1 1.257s
```
```go
// 호출하는 코드 2
m := memo.New(httpGetBody) 
var n sync.WaitGroup
for url := range incomingURLs() {

    n.Add(1) 
    go func(url string) { 
        start := time.Now() 
        value, err := m.Get(url) 
        if err != nil {
        	log.Print(err) 
        }
        fmt.Printf("%s, %s, %d bytes\n", url, time.Since(start), 
                    len(value.([]byte))) 
        n.Done()
	}(url)
	
}// end of for
n.Wait()

// 테스트 결과
$gotest -run=TestConcurrent -race -v gopl.io/ch9/memo1 
=== RUN TestConcurrent 
... 
WARNING: DATA RACE 
Write by goroutine 36: 
    runtime.mapassign1()
    ~/go/src/runtime/hashmap.go:411 +0x0 
    gopl.io/ch9/memo1.(*Memo).Get()
    ~/gobook2/src/gopl.io/ch9/memo1/memo.go:32 +0x205
    ...
Previous write by goroutine 35: 
    runtime.mapassign1()
    ptg16091132
    ~/go/src/runtime/hashmap.go:411 +0x0 
    gopl.io/ch9/memo1.(*Memo).Get() 
    ~/gobook2/src/gopl.io/ch9/memo1/memo.go:32 +0x205
    ... 
Found 1 data race(s) 
FAIL gopl.io/ch9/memo1 2.393s
```
// 2단계 : Sync.Mutex로 보완하여 데이터 경쟁 상태 해결
```go
// gopl.io/ch9/memo2 
type Memo struct { 
    f     Func 
    mu    sync.Mutex // cache를 보호하는 뮤텍스
    cache map[string]result
}
// 동시성에 안전하나, 모든 I/O를 순차적으로 변경 (블로킹)
func (memo *Memo) Get(key string) (value interface{}, err error) { 
    memo.mu.Lock() 
    res, ok := memo.cache[key] 
    if !ok { 
        res.value, res.err = memo.f(key) 
        memo.cache[key] = res 
    }
    memo.mu.Unlock()
    return res.value, res.err
}
```
// 3단계 : 캐시의 저장과 조회 분리로 논블로킹 구현
```go
//gopl.io/ch9/memo3 
func (memo *Memo) Get(key string) (value interface{}, err error) { 
    memo.mu.Lock() 
    res, ok := memo.cache[key] 
    memo.mu.Unlock() 
    if !ok { 
        res.value, res.err = memo.f(key)
        // 두개 이상의 고루틴이 저장되지 않은 URL값 처리 결과를 호출할 경우
        // f(key)가 2번 호출될 수 있으며, 값의 저장 또한 2번 일어날 수 있음.
        // 메모리 공유에 안전하지 않음
        memo.mu.Lock() 
        memo.cache[key] = res 
        memo.mu.Unlock() 
    }
    return res.value, res.err
}
```
// 4단계 : 논블로킹, 중복억제, 동시성 안전한 코드 구현
```go
//gopl.io/ch9/memo4

package memo

import "sync"

type Func func(string) (interface{}, error)

type result struct {
	value interface{}
	err   error
}

type entry struct {
	res   result
	ready chan struct{} // 추가 : result 저장이 끝났을때 표시
}
func New(f Func) *Memo {
	return &Memo{f: f, cache: make(map[string]*entry)}
}

type Memo struct {
	f     Func
	mu    sync.Mutex
	cache map[string]*entry
}

func (memo *Memo) Get(key string) (value interface{}, err error) {
	memo.mu.Lock()
	e := memo.cache[key]
	if e == nil {
		// 해당 키에 대한 최초 요청시 수행.
		// 요청 URL 처리후 준비 상태(ready) 설정
		e = &entry{ready: make(chan struct{})}
		memo.cache[key] = e
		memo.mu.Unlock()
		// 처리후 결과 저장
		e.res.value, e.res.err = memo.f(key)
		// ready 상태 브로드캐스팅
		close(e.ready)
	} else {
		// This is a repeat request for this key.
		memo.mu.Unlock()

		<-e.ready // ready 상태 설정 대기
	}
	return e.res.value, e.res.err
}

```

// 5단계 : Mutex없이 채널 사용한 예제
```go
//gopl.io/ch9/memo5

package memo

//!+Func 구조
type Func func(key string) (interface{}, error)
type result struct {
	value interface{}
	err   error
}
type entry struct {
	res   result
	ready chan struct{}
}
//!-Func 구조

//!+조회 처리

// request : Func를 key로 호출했을 떄의 결과를 요청하는 메시지
type request struct {
	key      string
	response chan<- result // the client wants a single result
}

type Memo struct{ requests chan request }
/*
//legacy code
type Memo struct {
	f     Func
	mu    sync.Mutex
	cache map[string]*entry
}
*/

// New : 캐시된 f의 결과 반환. 이후 반드시 Close호출
func New(f Func) *Memo {
	memo := &Memo{requests: make(chan request)}
	go memo.server(f)
	return memo
}

func (memo *Memo) Get(key string) (interface{}, error) {
	response := make(chan result)
	memo.requests <- request{key, response}
	res := <-response
	return res.value, res.err
}

func (memo *Memo) Close() { close(memo.requests) }

//!-조회 처리

//!+모니터

func (memo *Memo) server(f Func) {
	cache := make(map[string]*entry)
	for req := range memo.requests {
		e := cache[req.key]
		if e == nil {
			// 해당 키에 대한 최초 요청시 수행
			e = &entry{ready: make(chan struct{})}
			cache[req.key] = e
			go e.call(f, req.key) // 함수 처리 호출
		}
		go e.deliver(req.response) // 클라이언트 송신 처리 호출
	}
}

func (e *entry) call(f Func, key string) {
	// 함수 처리 호출
	e.res.value, e.res.err = f(key)
	// 함수 ready 상태 전파
	close(e.ready)
}

func (e *entry) deliver(response chan<- result) {
	// ready 상태 수신 대기
	<-e.ready
	// 클라이언트에 처리 결과 송신
	response <- e.res
}

//!-모니터

```

### 9.8  고루틴과 스레드의 차이

#### 9.8.1 가변 스택

- 각 OS의 스레드는 고정 크기의 스택(보통 2MB)
- 고루틴은 가변 크기의 스택 : 2KB에서 필요한 만큼 늘어나고 줄어든다.

#### 9.8.2 고루틴 스케쥴링

-  OS 스레드는 OS 커널에 의해 스케쥴링 : 하드웨어 타이머에 의해 주기적으로 전체 컨텍스트를 전환일어나며,각 스레드의 상태 메모리 저장 및 복원, 스케쥴러 자료 구조 갱신 등 고비용의 작업이 수행되며, CPU사이클수 증가 시킴

- 고루틴은 m:n 스케쥴러(m : 고루틴, n : os스레드) 사용. 

  go언어 기반에 의해 묵시적으로 호출되며, 단일GO 프로그램에 한정되어 스케쥴링 됨.

#### 9.8.3 GOMAXPROCS

-  동시에 수행할 OS 스레드 개수. GO 스케쥴러가 참조. 기본값은 시스템의 CPU 개수.
-  슬립 상태이거나 통신을 대기하고 있을때 고루틴은 스레드가 전혀 필요하지 않다.
-  I/O 또는 다른 시스템 호출 또는 GO가 아닌 함수 호출하는 쓰레드는 GOMAXPROCS에 포함시킬 필요 없음.

```go
1: for {
2:     go fmt.Print(0)
3:     fmt.Print(1)
4: }

// GOMAXPROCS=1인 경우 : 라인 2의 고루틴은 슬립 상태. 일정시간 후 깨어나 실행
$GOMAXPROCS=1 go run hacker-cliché.go 
111111111111111111110000000000000000000011111...
// GOMAXPROCS=2인 경우 : 스레드 2개 사용가능하므로, 라인2, 3 비슷한 비율로 수행
$GOMAXPROCS=2 go run hacker-cliché.go 
010101010101010101011001100101011010010100110...
```

#### 9.8.4 고루틴에는 식별자가 없다

- 고루틴에는 개발자가 접근 가능한 식별자에 대한 표현 방법이 없다.
  특정 스레드의 로컬스토리지가 남용되지 않도록 의도적으로 설계된 것.

  예를 들면 특정 스레드는 UI만 담당하도록 지정하는 행위 등

- Go는 함수의 동작에 영향 주는 파라메터를 명시하게 하는 단순한 프로그래밍을 권장한다.

- 추천하지 않지만 제공되는 API

  runtime.Stack ( https://golang.org/pkg/runtime/#Stack)
  출력된 문자를 파싱한후 첫번째 행에 ID가 포함되어 있다고 함.


