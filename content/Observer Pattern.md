%% [[Design Pattern]] %%
Observer Pattern（观察者模式）是对象的行为模式，核心是当一个对象的状态发生变化时，所有依赖它的对象都会自动收到通知并通知。
```Go
// 观察者
// 这里需要注意的是这个update的语意是，被观察对象变化了，观察者接收到通知，然后更新自己的状态或执行自己的逻辑。
type Observer interface {
	Update(message string)
}
```
```Go
// 被观察者
// Q1 为什么被观察者中有注册和取消注册和通知呢？
// A1 因为被观察者需要知道被哪些观察者观察，才能在状态变化的时候通知他们
type Subject interface {
	Register(observer Observer)
	Unregister(observer Observer)
	Notify(message string)
}
```
```Go
// 实现观察者
type NewChannel struct {
	name string
}

// 同步消息
func (c *NewChannel) Update(message string) {
	fmt.Printf("[%s] 收到新闻: %s\n", c.name, message)
}

// 实现被观察者
// 被观察者需要什么呢？需要知道有哪些观察者，这样可以通知
type NewAgency struct {
	observers []Observer
}

// 注册
func (n *NewAgency) Register(observer Observer) {
	n.observers = append(n.observers, observer)
}

// 取消注册
func (n *NewAgency) Unregister(observer Observer) {
	for i, item := range n.observers {
		if observer == item {
			n.observers = append(n.observers[:i], n.observers[i+1:]...)
			return
		}
	}
}

// 发消息
func (n *NewAgency) Notify(message string) {
	for _, item := range n.observers {
		item.Update(message)
	}
}


func main() {
	agency := &NewAgency{}
	
	tv := &NewChannel{name: "TV"}
	app := &NewChannel{name: "App"}

	agency.Register(tv)
	agency.Register(app)

	agency.Notify("Go 1.25 发布了")
	agency.Unregister(tv)

	agency.Notify("新的设计模式文章上线了")
}
```
应用场景：
1. 对一个对象状态或数据的更新需要其他对象同步更新，或者一个对象的更新需要依赖另一个对象的更新。
2. 对象仅需要将自己的更新通知给其他对象而不需要知道其他对象的细节，如消息推送。