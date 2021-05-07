# channelx
[![Go Report Card](https://goreportcard.com/badge/github.com/Ksloveyuan/channelx)](https://goreportcard.com/report/github.com/Ksloveyuan/channelx)
[![build](https://api.travis-ci.org/Ksloveyuan/channelx.svg?branch=master)](https://api.travis-ci.org/Ksloveyuan/channelx.svg?branch=master)
[![codecov](https://codecov.io/gh/Ksloveyuan/channelx/branch/master/graph/badge.svg)](https://codecov.io/gh/Ksloveyuan/channelx)
[![GoDoc](https://godoc.org/github.com/Ksloveyuan/channelx?status.svg)](https://godoc.org/github.com/Ksloveyuan/channelx)

Some useful tools implemented by channel to increase development efficiency, e.g. stream, promise, actor, parallel runner, aggregator, etc..

# blogs
- [如何把Golang的channel用的如nodejs的stream一样丝滑](https://juejin.im/post/5d7ba76ef265da03be490856)
- [如何用Golang的channel实现消息的批量处理](https://juejin.im/post/5d8c6775e51d45781332e91f)
- [如何把golang的Channel玩出async和await的feel](https://juejin.im/post/5e4175a36fb9a07ca80a9c77)
- [下次想在Golang中写个并发处理，就用这个模板，准没错！](https://juejin.cn/post/6959369791937347621/)

## Promise
A golang style async/await, even I call it Promise, while the api is not 100% aligns with Javascript Promise.

```golang
promise := NewPromise(func() (res interface{}, err error) {
    // do work asynchronously here
    reuturn
}).Then(func(input interface{}) (interface{}, error) {
    // here is the succss handler, which aslo runs asynchronously 
}, func(err error) interface{} {
    // here is the error handler, which aslo runs asynchronously 
})

// await: wait until it completes.
res, _ := promise.Done()
```
more examples, please check [promise_test.go](https://github.com/Ksloveyuan/channelx/blob/master/promise_test.go)

## Actor
The actor pattern is also called as Active Object, it seems like Promise, but the difference is Actor can be reused and it is FIFO.

```golang
actor := NewActor(SetActorBuffer(0))
defer actor.Close()

// do some work asynchroniously.
call := actor.Do(func() (interface{}, error) {
    time.Sleep(0 * time.Second)
    return 0, nil
})

// can to some other synchroniouse work here
// ......

// wait for the call completes.
res, err := call.Done()
```
more examples, please check [actor_test.go](https://github.com/Ksloveyuan/channelx/blob/master/actor_test.go)

## Stream
Steam works like Node.Js stream, it can be piped and data flows through the pipe one by one.

### before
```golang
var multipleChan = make(chan int, 4)
var minusChan = make(chan int, 4)
var harvestChan = make(chan int, 4)

defer close(multipleChan)
defer close(minusChan)
defer close(harvestChan)

go func() {
    for i:=1;i<=100;i++{
        multipleChan <- i
    }
}()

for i:=0; i<4; i++{
    go func() {
        for data := range multipleChan {
            minusChan <- data * 2
            time.Sleep(10* time.Millisecond)
        }
    }()

    go func() {
        for data := range minusChan {
            harvestChan <- data - 1
            time.Sleep(10* time.Millisecond)
        }
    }()
}

var sum = 0
var index = 0
for data := range harvestChan{
    sum += data
    index++
    if index == 100{
        break
    }
}

fmt.Println(sum)
```

### after

```golang
var sum = 0

NewChannelStream(func(seedChan chan<- Item, quitChannel chan struct{}) {
    for i:=1; i<=100;i++{
        seedChan <- Item{Data:i}
    }
    close(seedChan) //don't forget to close it
}).Pipe(func(Item Item) Item {
    return Item{Data: Item.Data.(int) * 2}
}).Pipe(func(Item Item) Item {
    return Item{Data: Item.Data.(int) - 1}
}).Harvest(func(Item Item) {
    sum += Item.Data.(int)
})

fmt.Println(sum)
```

more examples, please check [stream_test.go](https://github.com/Ksloveyuan/channelx/blob/master/stream_test.go)

## Aggregator
Aggregator is used for the scenario that receives request one by one while handle them in a batch would increase efficiency.

```golang
// YourKnownType, YourBatchHandler, yourRequest are faked type or object

batchProcess := func(items []interface{}) error {
    var arr YourKnownType 
    for _, item := range items{
        ykt := item.(YourKnownType)
        arr = append(arr, ykt)
    }
    
    YourBatchHandler(arr)
}

aggregator := NewAggregator(batchProcess)

aggregator.Start()

aggregator.Enqueue(yourRequest)

aggregator.Stop()
```
more examples, please check [aggregator_test.go](https://github.com/Ksloveyuan/channelx/blob/master/aggregator_test.go)