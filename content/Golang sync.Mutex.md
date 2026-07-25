%% [[Golang]] %%
Mutex是sync包里的一个结构体类型，含义是互斥锁。

Mutex变量的默认值（零值）是一个没有加锁的mutex，也就是说当前的mutex是unlock状态，不要对Mutex通过值传递的方式进行函数调用。

Mutex允许一个goroutine对其加锁，其他goroutine对其解锁，不要求加锁和解锁在同一个goroutine中。注意：Mutex不是递归锁，不可重入

- **加锁** `Lock()`会把Mutex变量m锁住，如果m已经锁住，在此调用此方法就会阻塞，直到锁释放。
- **解锁** `Unlock()`会把Mutex变量m解锁，如果m没有被锁，还调用此方法会遇到`runtime error`

以下先展示一个不加锁的例子，过个goroutine对共享变量同时执行写操作，并发是不安全的，结果与预期不符

```Go
// import ...

var sum int = 0

func add(i int) {
    sum += i
}

func main() {
    var wg sync.WaitGroup
    size := 100
    wg.Add(size)
    for i:=1; i<=size; i++ {
        i := i
        go func() {
            defer wg.Done()
            add(i)  // 操作全局变量sum
        }()
    }
    wg.Wait()
    fmt.Printf("sum of 1 to %d is: %d\n", size, sum)
}
```

以下的例子表示，通过对共享变量加互斥锁来保证并发安全，结果与预期相符，多个goroutine同时访问add，sum是多个goroutine共享的，通过加互斥锁来保证并发安全。

```Go
// import ...

var sum int = 0
var mutex sync.Mutex

func add(i int) {
    mutex.Lock()
    defer mutex.Unlock()
    sum += i
}

func main() {
    var wg sync.WaitGroup
    size := 100
    wg.Add(size)
    for i:=1; i<=size; i++ {
        i := i
        go func() {
            defer wg.Done()
            add(i)
        }()
    }
    wg.Wait()
    fmt.Printf("sum of 1 to %d is: %d\n", size, sum)
}
```
