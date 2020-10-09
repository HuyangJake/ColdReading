---
title: Go并发编程基础概念
date: 2020-10-09 10:17:07
tags: [Go]
---

## 协程机制

Thread vs. Groutine
1. 创建时默认的stack的大小
    * Groutine的Stack初始化大小为2K
2. 和KSE（Kernel Space Entity）的对应关系
    1. Java Thread 是 1 : 1
        1. 线程之间发生context切换的时候回直接造成内核对象的切换，造成很大的消耗
        ![-w1202](http://qiniu.huyangjie.cn/mweb/15939614397342.jpg)

    2. Groutine 是 M:N

<!-- more -->

### 协程原理图

![](http://qiniu.huyangjie.cn/mweb/15939616949160.jpg)

#### 结构
Processor为Go语言实现的协程处理器，Processor在不同的系统线程里，每个Processor有一个准备运行的协程队列，有一个正在运行的协程。

#### 并发性能
1. GO运行的时候会有一个守护线程进行计数，记录每个Processor的完成协程的数量。当一段时间完成的数量没有变化的时候，就会在该Processor的协程任务栈中插入一个特殊的标记，当协程运行非内联函数时就会遇到此标记，就会将自己中断下来然后插在等待队列的队尾，切换成别的协程继续运行。

2. 协程被中断的时候，Processor会把自己移到另一个可使用的系统线程中，继续执行所挂队列里的其他协程。当被中断的协程被唤醒时，它会被加在其中某一个Processor的等待队列中，或者全局等待队列中。其在寄存器中的状态会被保存到协程对象中，在下一次获取执行机会的时候，会再将状态保存到寄存器中。


### 使用

```go
func TestGroutine(t *testing.T) {
	for i := 0; i < 10; i++ {
		go func(i int) {
		//这里的i在每个协程中的地址是不一样的，会进行值复制
			fmt.Println(i)
		}(i)
		
		/*
        go func() {
		//这样写就会存在内存共享，会存在竞态
		  fmt.Println(i)
		}()
		*/
	}
	time.Sleep(time.Millisecond * 50)
}
```

## 控制并发

### 共享内存并发机制

#### 使用锁 `Mutex` 保证线程读写安全

``` go

func TestCounterThreadSafe(t *testing.T) {
	var mut sync.Mutex
	counter := 0
	for i := 0; i < 5000; i++ {
		go func() {
			defer func() {
				mut.Unlock()
			}()
			mut.Lock()
			counter++
		}()
	}
	time.Sleep(1 * time.Second)
	t.Logf("counter = %d", counter)
}
```

#### WaitGroup 可以进行同步操作

``` go 
func TestCounterWaitGroup(t *testing.T) {
	var mut sync.Mutex
	var wg sync.WaitGroup
	counter := 0
	for i := 0; i < 5000; i++ {
		wg.Add(1)
		go func() {
			defer func() {
				mut.Unlock()
			}()
			mut.Lock()
			counter++
			wg.Done()
		}()
	}
	wg.Wait()
	t.Logf("counter = %d", counter)
}
```

### CSP并发机制

依赖于一个通道来完成两个通讯实体之间的协调


#### Channel
典型的Channel机制， 通讯两方都需要在，传递和接受消息都需要

![](http://qiniu.huyangjie.cn/mweb/15943060683677.jpg)


Buffer Channel 发送者和接受者松耦合。channel会有一个容量，在没有满的情况下都可以发消息

![](http://qiniu.huyangjie.cn/mweb/15943060759561.jpg)


``` go

func service() string {
	time.Sleep(time.Millisecond * 500)
	return "Done"
}

func AsyncService() chan string {
	retCh := make(chan string)
	//设置了容量之后，就是buffer channel， 不会阻塞当前的协程
	// retCh := make(chan string, 1)
	go func() {
		ret := service()
		fmt.Println("returned result")
		retCh <- ret
		fmt.Println("service exited")
	}()
	return retCh
}

func TestAsyncService(t *testing.T) {
	retCh := AsyncService()
	otherTask()
	fmt.Println(<-retCh)
}
```

#### channel的关闭

* 向关闭的channel发送数据，会导致panic
* v , ok <- ch; ok 为 bool值，true 表示正常接收，false表示通道关闭
* 所有的channel 接受者都会在channel关闭时，立刻从阻塞等待中返回且上述ok值为false，数据会返回通道对应类型值的零值。

    >这个广播机制常被利用，进行向多个订阅者同时发送信号。如：退出信号
    
``` go
func dataProducer(ch chan int, wg *sync.WaitGroup) {
	go func() {
		for i := 0; i < 10; i++ {
			ch <- i
		}
		//可以同时跟很多订阅者发送特殊的通知
		close(ch)
		ch <- 11
		wg.Done()
	}()
}

func dataReceiver(ch chan int, wg *sync.WaitGroup) {
	go func() {
		for {
			if data, ok := <-ch; ok {
				fmt.Println(data)
			} else {
				break
			}
		}
		wg.Done()
	}()
}

func TestCloseChannel(t *testing.T) {
	var wg sync.WaitGroup
	ch := make(chan int)
	wg.Add(1)
	dataProducer(ch, &wg)
	wg.Add(1)
	dataReceiver(ch, &wg)
	wg.Add(1)
	dataReceiver(ch, &wg)
	wg.Wait()
}
```

### 多路选择和超时控制

![](http://qiniu.huyangjie.cn/mweb/15943074210765.jpg)


![](http://qiniu.huyangjie.cn/mweb/15943074336801.jpg)


``` go
func TestSelect(t *testing.T) {
	select {
	case ret := <-AsyncService():
		t.Log(ret)
	case <-time.After(time.Millisecond * 100):
		t.Error("time out")
	}
}

```

100毫秒的时候到了之后，AsyncService() 还没有返回就会进入超时的错误信息打印

### 任务取消

```go 
func isCancelled(cancelChan chan struct{}) bool {
	select {
	case <-cancelChan:
		return true
	default:
		return false
	}
}

//使用close广播机制进行批量取消任务
func cancel_2(cancelChan chan struct{}) {
	close(cancelChan)
}

func TestCancel(t *testing.T) {
	cancelChan := make(chan struct{}, 0)
	for i := 0; i < 5; i++ {
		go func(i int, cancelCh chan struct{}) {
			for {
				if isCancelled(cancelCh) {
					break
				}
				time.Sleep(time.Millisecond * 5)
			}
			fmt.Println(i, "Cancelled")
		}(i, cancelChan)
	}
	cancel_2(cancelChan)
	time.Sleep(time.Millisecond * 1)
}
```

### Context 上下文取消任务

* 根Context： 通过 context.Background()创建
* 子Context： context.WithCancel(parentContext)创建
    * ctx, cancel := context.WithCancel(context.Background())
* 当前Context被取消时，基于他的子context 都会被取消
* 接收消息通知<-ctx.Done()

``` go

func isCancelled(ctx context.Context) bool {
	select {
	case <-ctx.Done():
		return true
	default:
		return false
	}
}

func TestCancel(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	for i := 0; i < 5; i++ {
		go func(i int, ctx context.Context) {
			for {
				if isCancelled(ctx) {
					break
				}
				time.Sleep(time.Millisecond * 5)
			}
			fmt.Println(i, "Cancelled")
		}(i, ctx)
	}
	cancel()
	time.Sleep(time.Millisecond * 1)
}

```