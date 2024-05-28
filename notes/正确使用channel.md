# 正确使用channel

## channel的特性

* channel会阻塞输入或者输出端，直到有goroutine接收或者写入
* channel只能由produce端close（调用close函数），输出端close编译不通过（必须要有produce的权限）
* channel能且只能close一次，第二次会报panic
* channel关闭之后不能写入，写入会报panic
* channel的接受端可以有两个返回值，如果第二个返回值为false，代表channel关闭
* 已关闭的channel也可用读取，但是标志位（第二个返回值）会是false

## 使用场景

### spsc

spsc（一个生产者一个消费者）：

这是理想情况，只需要produce端close即可

```go
func send(ch chan<- int) {
    ch <- 1
    close(ch)
}

func consume(ch <-chan int) {
    num := <-ch
    fmt.Println(num)
}
```

### spmc

spmc（一个生产者多个消费者）：

比较简单常见的情况，也是只需要produce端close，但是consume端需要判断标志位（否则使用的是零值，可能会造成错误）

```go
func send(ch chan<- int) {
    ch <- 1
    close(ch)
}

func consume(ch <-chan int) {
    num ,ok:= <-ch
    if ok{
        fmt.Println(num)
    }
}
```

### mpsc

mpsc（多个生产者一个消费者）：

略微复杂的情况，这里需要由consume端负责close（意味着consume需要使用`chan`，而不是`<-chan`），同时produce端需要使用recover()处理panic

```go
func send(ch chan<- int) {
    defer func() {
        if err := recover();err!=nil{
            fmt.Println(err)
        }
    }()
    ch <- 1
}

func consume(ch chan int) {
    num ,ok:= <-ch
    if ok{
        fmt.Println(num)
    }
    close(ch)
}
```

这个方法还是不太优雅，最好使用两个channel(或者是sync.waitGroup,在另一个线程中关闭)，使用另一个channel通知producer不要写入了，有多少个producer就发送多少条，在最后一个producer关闭channel

```go
func send(ch chan<- int,flag <-chan bool) {
    for {
        select {
        case f, _ := <-flag:
            if f == 3 {
                close(ch)
                fmt.Println("successes closed")
                return
            }
            fmt.Println("closed")
            return
        case ch <- 1:
            fmt.Println("send")
        }
    }
}

func consume(ch <-chan int,flag chan<- bool) {
    for message := range ch {
        fmt.Println("received :",message)
        for i := 0; i < 4; i++ {
            flag<-i
        }
        close(flag)
    }
}
```

### mpmc

mpmc（多个生产者多个消费者）：

最复杂的情况，这种场景下produce和consume端都可能会close，具体视业务场景而定。此时produce端需要处理panic，consume端需要处理标志位和panic

```go
func send(ch chan<- int) {
    defer func() {
        if err := recover();err!=nil{
            fmt.Println(err)
        }
    }()
    ch <- 1
    //close(ch)
}
func consume(ch chan int) {
    defer func() {
        if err := recover();err!=nil{
            fmt.Println(err)
        }
    }()
    num ,ok:= <-ch
    if ok{
        fmt.Println(num)
    }
    //close(ch)
}
```

## 总结：

```
* produce端只给`chan<-`权限即可
* consume端如果需要close给`chan`，确定不需要给`<-chan`，最好使用for来取数据
* produce端最好都做recover处理（具体使用时可以`fmt.Sprint(err)=="send on closed channel"`来再次上抛不属于channel的panic）
* consume端最好都处理标志位，如果有写权限(`chan`)，可能也要处理panic
* 最后一条：尽量手动关闭channel
```

多生产者多消费者，最好使用辅助channel通知生产者停止消费数据，可以使用另一个goroutine或者是最后一个生产者关闭channel

channel的关闭原则：

* 不要在消费者端关闭channel
* 不要在有多个并行的生产者时关闭channel
* 应该只在唯一或者最后一个生产者协程中关闭channel

## PS

time包下有两个特殊的channel，使用After和Tick来实现定时
