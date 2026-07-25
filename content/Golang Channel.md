%% [[Golang]] %%
Channel 是 Go 中用于在 Goroutine 之间传递数据和实现同步的工具。它所体现的核心思想是：
>不要通过共享内存来通信，而应该通过通信来共享内存。
# 声明
```Go
var ch chan int
```
上述代码声明了一个只能传递 `int` 类型数据的 Channel。此时 Channel 还没有被初始化，其默认值为 `nil`，因此不能直接用于数据的发送和接收。
# 创建
```Go
ch := make(chan int)
```
make用于初始化Channel。这里创建的是一个只能传递int类型数据的Channel。由于没有指定容量，因此该Channel是一个无缓冲Channel。
# 无缓冲
创建无缓冲channel，无缓冲channel没有存储数据的空间，发送和接收数据需要同时准备好，数据才能传递，无缓冲channel不只是可以传递数据，还可以进行同步。
```Go
ch := make(chan int)  // 创建

go func() { // 发送数据
	// ...
	ch <- 100
}()

value := <-ch // 接收数据
```
以下代码会出现死锁的情况，发送方必须等待另一个Goroutine接收数据，但是当前程序只有主Goroutine，并且主Goroutine已经阻塞在发送操作上，因此后面的接收操作永远无法执行。让发送方和接收方在不同的Goroutine中。
```Go
func main() {
	ch := make(chan int)
	ch <- 10 
	value := <-ch
	// ...
}
```
在以下代码中，等待 2 秒的作用是为子 Goroutine 预留足够的执行时间，确保其中的输出或其他后续操作能够完成。

程序启动后，主 Goroutine 创建一个子 Goroutine。子 Goroutine 会阻塞在 Channel 的读取操作上，直到主 Goroutine 将数据写入 Channel。数据传递完成后，子 Goroutine 继续执行输出或其他逻辑，主 Goroutine 也会继续向下执行。

如果主 Goroutine 在子 Goroutine 完成后续操作之前结束，整个程序会直接退出，尚未执行完成的子 Goroutine 也会随之终止。因此，这里通过 `time.Sleep` 暂时阻塞主 Goroutine，避免程序过早退出。
```Go

type Cat struct {}

func fetchChannel(ch chan Cat) {
    value := <- ch
	// output ...
}


func main() {
    ch := make(chan Cat)
    a := Cat{...}
    
    go fetchChannel(ch)
    
    ch <- a
    
    // main这个goroutine在这里等待2秒
    time.Sleep(2*time.Second)
    fmt.Println("end")
}
```
不过，使用固定时长来等待子 Goroutine 执行完成并不是一种可靠的同步方式。[[Golang sync.WaitGroup|sync.WaitGroup]]
```Go
type Cat struct{}

func fetchChannel(ch chan Cat, wg *sync.WaitGroup) {
	defer wg.Done()
	value := <-ch
	fmt.Println(value)
	// 其他后续操作
}

func main() {
	ch := make(chan Cat)
	var wg sync.WaitGroup
	wg.Add(1)
	a := Cat{}
	go fetchChannel(ch, &wg)
	ch <- a
	// 等待子 Goroutine 执行完成
	wg.Wait()
	fmt.Println("end")
}
```
这里需要注意：对于无缓冲 Channel，`ch <- a` 会一直阻塞，直到子 Goroutine 接收到数据。但是，**接收到数据并不代表子 Goroutine 的后续逻辑已经执行完成**。

因此，Channel 负责 Goroutine 之间的数据传递，而 `WaitGroup` 负责等待子 Goroutine 执行结束。相比 `time.Sleep`，这种方式不会依赖机器性能或任务执行时间，是一种更加可靠、明确的同步方式。
# 有缓冲
创建带有缓冲区的channel，这里表示最多可以存储的元素个数
```Go
ch := make(chan int, 3)
```
对于有缓冲的channel来说，以下几种情况会导致阻塞的发生：
- 缓冲区已满继续让发送者发送
- 缓冲区为空继续让接收者接收
# 方向
默认情况下，channel 可以同时发送和接收：`chan int`，也可以限制 channel 的方向。
- 只发送 channel：`chan<- int`
- 只接收 channel：`<-chan int`
# 遍历
# 关闭操作
使用内置函数`Close()`关闭channel，关闭channel表示后续不会再有数据进入，并且通常由发送方关闭channel。
```Go
func producer(ch chan<- int) { 
	for i := 0; i < 3; i++ { 
		ch <- i 
	} 
	close(ch) 
}
```
接收方可以继续读取channel中已经存在的数据。
```Go
ch := make(chan int, 3)

ch <- 1
// 不断写入...

close(ch)

<-ch
// 不断读取，直到读取完全部内容
```
判断channel是否关闭，可以用以下方法，接收操作可以返回两个值：
```Go
value, ok := <-ch
```
其中：
- value是接收到的数据；
- ok为true表示成功读取到正常数据；
- ok为false表示channel已关闭，并且缓冲区已经为空。
关闭操作的注意事项，在关闭之后：
- 如果再往channel里发送数据，会引发panic
- 如果再次close，也会引发panic
- 如果channel还有值，接收方可以一直从channel里获取值，直到channel里的值都已经取完。
- 如果channel里没有值了，接收方继续从channel里取值，会得到channel里存的数据类型对应的默认零值，如果一直取值，就一直拿到零值。

