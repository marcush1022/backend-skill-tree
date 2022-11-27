# **go 语言坑之并发访问 map**

- map 的使用有一定的限制
    
    - 如果是在单个协程中读写 map，那么不会存在什么问题

    - **如果是多个协程并发访问一个 map，有可能会导致程序退出**，并打印下面错误信息：

        ```bash
        fatal error: concurrent map read and map write
        ```

- **上面的这个错误不是每次都会遇到的，如果并发访问的协程数不大，遇到的可能性就更小了**。

- 例如下面的程序：

    ```go
    package main

    func main() {
        Map := make(map[int]int)

        for i := 0; i < 10; i++ {
            go writeMap(Map, i, i)
            go readMap(Map, i)
        }

    }

    func readMap(Map map[int]int, key int) int {
        return Map[key]
    }

    func writeMap(Map map[int]int, key int, value int) {
        Map[key] = value
    }
    ```

- 只循环了 10 次，产生了 20 个协程并发访问 map，程序基本不会出错，但是如果将循环次数变大，比如 10 万，运行下面程序基本每次都会出错：

    ```go
    package main

    func main() {
        Map := make(map[int]int)

        for i := 0; i < 100000; i++ {
            go writeMap(Map, i, i)
            go readMap(Map, i)
        }

    }

    func readMap(Map map[int]int, key int) int {
        return Map[key]
    }

    func writeMap(Map map[int]int, key int, value int) {
        Map[key] = value
    }
    ```

    ```bash
    fatal error: concurrent map writes

    goroutine 7 [running]:
    runtime.throw(0x107517d, 0x15)
        /usr/local/Cellar/go/1.14.3/libexec/src/runtime/panic.go:1116 +0x72 fp=0xc00003d758 sp=0xc00003d728 pc=0x1029892
    runtime.mapassign_fast64(0x1063e20, 0xc000060000, 0x1, 0x0)
        /usr/local/Cellar/go/1.14.3/libexec/src/runtime/map_fast64.go:101 +0x323 fp=0xc00003d798 sp=0xc00003d758 pc=0x100d643
    main.writeMap(0xc000060000, 0x1, 0x1)
        /Users/neo/tmp/go.go:18 +0x41 fp=0xc00003d7c8 sp=0xc00003d798 pc=0x1057a71
    runtime.goexit()
        /usr/local/Cellar/go/1.14.3/libexec/src/runtime/asm_amd64.s:1373 +0x1 fp=0xc00003d7d0 sp=0xc00003d7c8 pc=0x1053491
    created by main.main
        /Users/neo/tmp/go.go:7 +0x5f

    goroutine 1 [runnable]:
    runtime.gopark(0x0, 0x0, 0xc000051008, 0x1)
        /usr/local/Cellar/go/1.14.3/libexec/src/runtime/proc.go:287 +0x130
    runtime.main()
        /usr/local/Cellar/go/1.14.3/libexec/src/runtime/proc.go:222 +0x28c
    runtime.goexit()
        /usr/local/Cellar/go/1.14.3/libexec/src/runtime/asm_amd64.s:1373 +0x1

    goroutine 17 [runnable]:
    main.writeMap(0xc000060000, 0x6, 0x6)
        /Users/neo/tmp/go.go:17
    created by main.main
        /Users/neo/tmp/go.go:7 +0x5f
    ```

- go官方博客有如下说明：

    > Maps are not safe for concurrent use: **it's not defined what happens when you read and write to them `simultaneously`**. 
    > 
    > If you need to read from and write to a map from concurrently executing goroutines, **the accesses must be mediated by some kind of `synchronization mechanism`**. 
    > 
    > **One common way to protect maps is with `sync.RWMutex`**.

- 并发访问 map 是不安全的，会出现未定义行为，导致程序退出。

- 所以如果希望在多协程中并发访问 map，必须提供某种同步机制，**一般情况下通过读写锁 `sync.RWMutex` 实现对 map 的并发访问控制**，将 map 和 sync.RWMutex 封装一下，可以实现对 map 的安全并发访问

- 示例代码如下：

    ```go
    package main

    import "sync"

    type SafeMap struct {
        sync.RWMutex
        Map map[int]int
    }

    func main() {
        safeMap := newSafeMap(10)

        for i := 0; i < 100000; i++ {
            go safeMap.writeMap(i, i)
            go safeMap.readMap(i)
        }

    }

    func newSafeMap(size int) *SafeMap {
        sm := new(SafeMap)
        sm.Map = make(map[int]int)
        return sm

    }

    func (sm *SafeMap) readMap(key int) int {
        sm.RLock()
        value := sm.Map[key]
        sm.RUnlock()
        return value
    }

    func (sm *SafeMap) writeMap(key int, value int) {
        sm.Lock()
        sm.Map[key] = value
        sm.Unlock()
    }
    ```

- 但是通过读写锁控制 map 的并发访问时，会导致一定的性能问题，不过能保证程序的安全运行，牺牲点性能问题是可以的。

