a rendezvous point for goroutines waiting for or announcing the occurrence of an event
#### cond
goroutine 不消耗资源的情况下睡眠，直到收到唤醒信号
```go
	c := sync.NewCond(&sync.Mutex{})
	c.L.Lock()
	for conditionTrue() == false {
		c.Wait()
	}
	c.L.Unlock()
```
1. c.wait() 阻塞并暂停 goroutine 直到收到通知
2. 调用 wait 时，进入等待状态，cond 变量将会调用 UnLock；离开等待状态时，会调用 Lock
```go
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}
```
#### 示例
两个 goroutine，一个等待信号一个发送信号，有一个固定长度的队列，长度2，有10个元素需要放入，并且一有空间就放入 
```go
	c := sync.NewCond(&sync.Mutex{})
	queue := make([]interface{}, 0, 10)
	removeFromQueue := func(delay time.Duration) {
		time.Sleep(delay)
		c.L.Lock()
		queue = queue[1:]
		fmt.Println("Removed from queue")
		c.L.Unlock()
		c.Signal() // 1
	}
	for i := 0; i < 10; i++{
		c.L.Lock()
		for len(queue) == 2 {
			c.Wait() // 2
		}
		fmt.Println("Adding to queue")
		queue = append(queue, struct{}{})
		go removeFromQueue(1*time.Second)
		c.L.Unlock()
	}
```
1. 处理完一个元素后发送信号
2. wait 只要收到信号就继续执行，而不在乎收到了什么