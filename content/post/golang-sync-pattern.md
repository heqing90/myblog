---
title: "Golang 两种同步变量的实现方式"
date: 2020-01-09T15:18:00+08:00
draft: true
tags :
    - "Golang"
    - "mutex"
    - "channel"
categories:
    - "入门"
---

## 使用 *sync.RWMutex* 同步数据读写

```go
    type syncData struct {
        sync.RWMutex
        val int
    }

    func (data *syncData) getValue() int {
        data.RLock()
        defer data.RUnlock()
        return data.val
    }

    func (data *syncData) setValue(v int) {
        data.Lock()
        defer data.Unlock()
        data.val = v
    }
```

## 使用 *channel* 同步数据读写

```go
    type chData struct {
        getCh chan int
        setCh chan int
        val   int
    }

    func newChData() *chData {
        d := &chData{getCh: make(chan int), setCh: make(chan int), val: 0}
        go d.mix()
        return d
    }

    func (data *chData) mix() {
        for {
            select {
            case data.getCh <- data.val:
            case data.val = <-data.setCh:
            }
        }
    }

    func (data *chData) getValue() int {
        return <-data.getCh
    }

    func (data *chData) setValue(v int) {
        data.setCh <- v
    }
```

## 对比测试

1. 测试代码

    ```go
        func BenchmarkChannelCompete(b *testing.B) {
            d := newChData()
            var wg sync.WaitGroup

            wg.Add(2)
            go func() {
                defer wg.Done()
                for i := 0; i < b.N; i++ {
                    d.getValue()
                }
            }()

            go func() {
                defer wg.Done()
                for i := 0; i < b.N; i++ {
                    d.setValue(i)
                }
            }()

            wg.Wait()
        }

        func BenchmarkMutexCompete(b *testing.B) {
            d := &syncData{}
            var wg sync.WaitGroup

            wg.Add(2)
            go func() {
                defer wg.Done()
                for i := 0; i < b.N; i++ {
                    d.getValue()
                }
            }()

            go func() {
                defer wg.Done()
                for i := 0; i < b.N; i++ {
                    d.setValue(i)
                }
            }()

            wg.Wait()
        }
    ```

2. 测试结果

    ```shell
        PS C:\Users\admin\go\src\test\go_sync> go test -benchmem -benchtime=5s test\go_sync -bench . -cpu=1
        goos: windows
        goarch: amd64
        BenchmarkChannelCompete         10000000              1069 ns/op               0 B/op          0 allocs/op
        BenchmarkMutexCompete           20000000               565 ns/op               0 B/op          0 allocs/op
        PASS
        ok      test/go_sync    23.764s
        PS C:\Users\admin\go\src\test\go_sync> go test -benchmem -benchtime=5s test\go_sync -bench . -cpu=2
        goos: windows
        goarch: amd64
        BenchmarkChannelCompete-2        5000000              1576 ns/op               0 B/op          0 allocs/op
        BenchmarkMutexCompete-2         10000000               942 ns/op               0 B/op          0 allocs/op
        PASS
        ok      test/go_sync    19.838s
        PS C:\Users\admin\go\src\test\go_sync> go test -benchmem -benchtime=5s test\go_sync -bench . -cpu=4
        goos: windows
        goarch: amd64
        BenchmarkChannelCompete-4        5000000              1496 ns/op               0 B/op          0 allocs/op
        BenchmarkMutexCompete-4          5000000              1222 ns/op               0 B/op          0 allocs/op
        PASS
        ok      test/go_sync    16.486s
    ```

## 小结

1. ***Mutex*** 方式实现数据的读写同步更为简洁方便，并且在一般情况下效率更高。
2. ***channel*** 方式实现数据的读写同步写法上略微复杂， 但是更容易处理异常情况，  
    比如在 *select* 时可以监听超时消息，进而做一些特殊处理。