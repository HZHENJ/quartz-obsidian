%% [[Golang]] %%
Map是sync包里的一个结构体类型，定义如下：

```Go
type Map struct {
    // some fields
}
```

在Go中普通的map的读写不是并发安全的，sync.Map的读写是并发安全的

`sync.Map`可以理解为一个类似`map[interface{}]interface{}`的结构，可以的类型可以不一样，`value`也可以类型不一样，多个`goroutine`对其进行读写不需要额外加锁。

Go官方设计`sync.Map`主要满足以下2个场景的用途：

1. 每个`key`只写一次，其它对该`key`的操作都是读操作
2. 多个`goroutine`同时读写`map`，但是每个`goroutine`只读写各自的`keys`

以上2种场景，相对于对普通的map加Mutex或者RWMutex来实现并发安全，使用sync.Map不用在业务代码里加锁，会大幅减少锁竞争，提升性能。其它更为常见的场景还是使用普通的Map，搭配Mutex或者RWMutex来使用。

不能对sync.Map使用值传递方式进行函数调用。

- **Delete** 删除map里的key，即使key不存在，执行Delete操作也没任何影响
- **Load** 从map里取出key对应的value。如果key存在map里，返回值value就是对应的值，ok就是true。如果key不在map里，返回值value就是nil，ok就是false。
- **LoadAndDelete** 删除map里的key。如果key存在map里，返回值value就是对应的值，loaded就是true。如果key不在map里，返回值value就是nil，loaded就是false。
- **LoadOrStore** 从map里取出key对应的value。如果key在map里不存在，就把LoadOrStrore函数调用传入的参数<key, value>存储到map里，并返回参数里的value。如果key在map里，那loaded是true，如果key不在map里，那loaded是false。
- **Range**
- **Store**

Delete, Load, LoadAndDelete, LoadOrStore, Store的均摊时间复杂度是O(1)，Range的时间复杂度是O(N)

初始化
```Go
var m1 sync.Map
m2 := sync.Map{}
```

统计字符串里每个字符出现的次数
```Go
// import ...

func main() {
    /*统计字符串里每个字符出现的次数*/
    m := sync.Map{}
    str := "abcabcd"
    for _, value := range str {
        temp, ok := m.Load(value)
        //fmt.Println(temp, ok)
        if !ok {
            m.Store(value, 1)
        } else {
            /*temp是个interface变量，要转int才能和1做加法*/
            m.Store(value, temp.(int)+1)
        }
    }
    
    /*使用sync.Map里的Range遍历map*/
    m.Range(func(key, value interface{}) bool{
        fmt.Println(key, value)
        return true
    })
}
```

多个goroutine并发写sync.Map，不加锁。如果是普通的map，这么来写就会出现运行时错误“fatal error: concurrent map writes”
```Go
// import ... 

var m sync.Map

/*
sync.Map里每个key只写一次，属于场景1
*/
func changeMap(key int) {
    m.Store(key, 1)
}

func main() {
    var wg sync.WaitGroup
    size := 2
    wg.Add(size)
    
    for i:=0; i<size; i++ {
        i := i
        go func() {
            defer wg.Done()
            changeMap(i)
        }()
    }
    wg.Wait()
    
    /*使用sync.Map里的Range遍历map*/
    m.Range(func(key, value interface{}) bool{
        fmt.Println(key, value)
        return true
    })
}
```
- sync.Map不支持len和cap函数
- 在评估是否要使用sync.Map的时候，先要考察业务场景是否符合上面描述的场景1和场景2在考虑是否用sync.Map不符合就用普通的map+Mutex或者RWMutex

