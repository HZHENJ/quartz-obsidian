# 声明
在Go中，interface和Java/C++中的interface有一个很大的区别：在Go中interface是“隐式实现（implicit implementation）“。在Java中，如果一个类想要实现某个接口，必须显示声明：
```Java
class WechatAdapter implements Payment
```
但在Go中，并不需要这样声明，Go会在编译阶段自动给检查某个类型是否实现了interface中所要求的全部方法，只要方法全部都实现了，那么这个类就自动满足（satisfy）这个interface。例如：有一个Payment的interface，其中有Pay方法，如下：
```Go
type Payment interface {
	Pay(int64)
}
```
这个interface表示，任何具有Pay方法的类型，都可以被当作Payment使用。假设现在接入微信支付的SDK：
```Go
type WechatSDK struct {}

func (w *WechatSDK) WechatPay(amount int64) {
	// wechat payment logic
}

// 可能存在的其他第三方SDK
```
这里有个问题，微信提供的支付接口是WechatPay，而当我们自己的系统的支付接口是Pay，那么就需要写一个针对微信支付接口的WechatAdapter适配器，这里就是设计模式中的[[Adapter Pattern|适配器模式]]。
```Go
// 适配器
type WechatAdapter struct {
	wechat *WechatSDK
}

func (w *WechatAdapter) Pay(amount int64) { 
	w.wechat.WechatPay(amount) 
}
```
这里WechatAdapter实现了`Pay(amount int64)`因此WechatAdapter自动满足Payment，Go的编译器会自动识别，这意味这我们在使用的时候可以这样用：
```Go
var p Payment
p = &WechatAdapter{}
```
Go 的 interface 更关注的是“行为（behavior）”而不是“类型继承关系”，也就是说Go 不关心这个类型是谁的子类，只关心这个类型有没有对应的方法，因此interface 本质上是一种“行为约束”。
---未完成---
- 多接口
- 接口作为参数
- 接口嵌套
- 空接口
- 断言