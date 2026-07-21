Goroutine之间传输数据和进行同步的工具。
核心思想：不要通过共享内存来通信，而应该通过通信来共享内存。

Channel的声明与创建
声明channel，初始值为nil。
```Go
var ch chan int
```
创建channel，这里表示channel只能传递int类型的数据。
```Go
ch := make(chan int)
```

无缓冲Channel
创建无缓冲channel，无缓冲channel没有存储数据的空间，发送和接收数据需要同时准备好，数据才能传递，无缓冲channel不只是可以传递数据，还可以进行同步（以下这段代码是否正确？
```Go
ch := make(chan int)  // 创建

go func() { // 发送数据
	// ...
	ch <- 100
}()

value := <-ch // 接收数据
```
出现死锁的情况，发送方必须等待另一个Goroutine接收数据，但是当前程序只有主Goroutine，并且主Goroutine已经阻塞在发送操作上，因此后面的接收操作永远无法执行。让发送方和接收方在不同的Goroutine中。
```Go
func main() {
	ch := make(chan int)
	ch <- 10 
	value := <-ch
	// ...
}
```

有缓冲Channel
创建带有缓冲区的channel，这里表示最多可以存储的元素个数
```Go
ch := make(chan int, 3)
```

关闭Channel