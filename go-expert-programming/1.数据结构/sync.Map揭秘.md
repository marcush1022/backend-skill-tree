# **sync.Map 揭秘**
- 在 Go 1.6 之前， 内置的 map 类型是部分 goroutine 安全的，并发的读没有问题，并发的写可能有问题。自 go 1.6 之后，并发地读写 map 会报错，所以 go 1.9 之前的解决方案是额外绑定一个锁，封装成一个新的 struct 或者单独使用锁都可以。

<br>

# **1. 有并发问题的 map**
- **官方的 faq 已经提到内建的 map 不是线程 (goroutine) 安全的**。

- 首先，让我们看一段并发读写的代码,下列程序中一个 goroutine 一直读，一个 goroutine 一只写同一个键值，即即使读写的键不相同，而且 map 也没有 "扩容" 等操作，代码还是会报错。

    ```go
    package main
    func main() {
        m := make(map[int]int)
        go func() {
            for {
                _ = m[1]
            }
        }()

        go func() {
            for {
                m[2] = 2
            }
        }()
        
        select {}
    }
    ```

- 错误信息是: `fatal error: concurrent map read and map write`。

- **如果你查看 Go 的源代码: `hashmap_fast.go#L118`，会看到读的时候会检查 `hashWriting` 标志，如果有这个标志，就会报`并发错误`**。

- 写的时候会设置这个标志: `hashmap.go#L542`

    ```go
    h.flags |= hashWriting
    ```

- `hashmap.go#L628` 设置完之后会取消这个标记。

- 当然，代码中还有好几处并发读写的检查，比如写的时候也会检查是不是有并发的写，删除键的时候类似写，遍历的时候并发读写问题等。

- **有时候，map 的并发问题不是那么容易被发现, 你可以利用 `-race` 参数来检查**。

<br>

# **2. Go 1.9 之前的解决方案**
- 但是，很多时候，我们会并发地使用 map 对象，尤其是在一定规模的项目中，**map 会保存 goroutine 共享的数据**。在 Go 官方 blog 的 Go maps in action 一文中，提供了一种简便的解决方案。

    ```go
    var counter = struct{
        sync.RWMutex
        m map[string]int
    }{m: make(map[string]int)}
    ```

- **它使用嵌入 struct 为 map 增加一个读写锁**。

- 读数据的时候很方便的加锁：

    ```go
    counter.RLock()
    n := counter.m["some_key"]
    counter.RUnlock()
    fmt.Println("some_key:", n)
    ```

- 写数据的时候:

    ```go
    counter.Lock()
    counter.m["some_key"]++
    counter.Unlock()
    ```

<br>

# **3. sync.Map**
- 可以说，上面的解决方案相当简洁，**并且利用`读写锁`而不是 `Mutex` 可以进一步减少读写的时候因为锁带来的性能影响**。

- 但是，它在一些场景下也有问题，如果熟悉 Java 的同学，可以对比一下 java 的 `ConcurrentHashMap` 的实现，**在 map 的数据非常大的情况下，一把锁会导致大并发的客户端共争一把锁**，Java 的解决方案是 `shard`, 内部使用`多个锁`，每个`区间共享一把锁`，这样减少了数据共享一把锁带来的性能影响。

- 那么，在 Go 1.9 中 sync.Map 是怎么实现的呢？它是如何解决并发提升性能的呢？

- sync.Map 的实现有几个优化点，这里先列出来，我们后面慢慢分析。

    1. **空间换时间**。通过冗余的两个数据结构 `(read、dirty)`，实现加锁对性能的影响。

    2. **使用只读数据 (read)，避免读写冲突**。

    3. **动态调整，miss 次数多了之后，将 dirty 数据提升为 read**。
    
    4. **double-checking**。
    
    5. **延迟删除。删除一个键值只是打标记，只有在提升 dirty 的时候才清理删除的数据**。
    
    6. **优先从 read 读取、更新、删除，因为对 read 的读取不需要锁**。

- sync.Map 的数据结构：

    ```go
    type Map struct {
        // 当涉及到 dirty 数据的操作的时候，需要使用这个锁
        mu Mutex

        // 一个只读的数据结构，因为只读，所以不会有读写冲突。
        // 所以从这个数据中读取总是安全的。
        // 实际上，实际也会更新这个数据的 entries,如果 entry 是未删除的 (unexpunged), 并不需要加锁。如果 entry 已经被删除了，需要加锁，以便更新 dirty 数据。
        read atomic.Value // readOnly

        // dirty 数据包含当前的 map 包含的 entries,它包含最新的 entries (包括 read 中未删除的数据,虽有冗余，但是提升 dirty 字段为 read 的时候非常快，不用一个一个的复制，而是直接将这个数据结构作为 read 字段的一部分)，有些数据还可能没有移动到 read 字段中。
        // 对于 dirty 的操作需要加锁，因为对它的操作可能会有读写竞争。
        // 当 dirty 为空的时候，比如初始化或者刚提升完，下一次的写操作会复制 read 字段中未删除的数据到这个数据中。
        dirty map[interface{}]*entry

        // 当从 Map 中读取 entry 的时候，如果 read 中不包含这个 entry，会尝试从 dirty 中读取，这个时候会将 misses 加一，
        // 当 misses 累积到 dirty 的长度的时候，就会将 dirty 提升为 read，避免从 dirty 中 miss 太多次。因为操作 dirty 需要加锁。
        misses int
    }
    ```

- 它的数据结构很简单，值包含四个字段：`read、mu、dirty、misses`。

- **它使用了`冗余的`数据结构 `read、dirty`。`dirty` 中会包含 `read` 中未删除的 `entries`，新增加的 `entries` 会加入到 `dirty` 中**。

- read 的数据结构是：

    ```go
    type readOnly struct {
        m       map[interface{}]*entry
        amended bool // 如果 Map.dirty 有些数据不在 m 中的时候，这个值为 true
    }
    ```

- `amended` 指明 `Map.dirty` 中有 `readOnly.m` 未包含的数据，**所以如果从 `Map.read` 找不到数据的话，还要进一步到 `Map.dirty` 中查找**。

- **对 `Map.read` 的修改是通过原子操作进行的**。

- **虽然 `read` 和 `dirty` 有冗余数据，但这些数据是`通过指针指向同一个数据`，所以尽管 `Map` 的 `value` 会很大，但是`冗余的空间占用`还是有限的**。

- `readOnly.m` 和 `Map.dirty` 存储的值类型是 `*entry`，它包含一个指针 p，指向用户存储的 value 值。

    ```go
    type entry struct {
        p unsafe.Pointer // *interface{}
    }
    ```

- p 有三种值：
    
    1. **`nil`: `entry` 已被删除了，并且 `m.dirty` 为 nil**
    
    2. **`expunged`: `entry` 已被删除了，并且 `m.dirty` 不为 nil，而且这个 `entry` 不存在于 `m.dirty` 中**
    
    3. **其它：entry 是一个正常的值**

- 以上是 `sync.Map` 的数据结构，下面我们重点看看 `Load、Store、Delete、Range` 这四个方法，其它辅助方法可以参考这四个方法来理解。

<br>

## **3.1. Load**
- 加载方法，也就是提供一个键 key，查找对应的值 value，如果不存在，通过 `ok` 反映：

    ```go
    func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
        // 1. 首先从 m.read 中得到只读 readOnly，从它的 map 中查找，不需要加锁
        read, _ := m.read.Load().(readOnly)
        e, ok := read.m[key]

        // 2. 如果没找到，并且 m.dirty 中有新数据，需要从 m.dirty 查找，这个时候需要加锁
        if !ok && read.amended {
            m.mu.Lock()
            // 双检查，避免加锁的时候 m.dirty 提升为 m.read，这个时候 m.read 可能被替换了。
            read, _ = m.read.Load().(readOnly)
            e, ok = read.m[key]

            // 如果 m.read 中还是不存在，并且 m.dirty 中有新数据
            if !ok && read.amended {
                // 从 m.dirty 查找
                e, ok = m.dirty[key]

                // 不管 m.dirty 中存不存在，都将 misses 计数加一
                // missLocked() 中满足条件后就会提升 m.dirty
                m.missLocked()
            }
            m.mu.Unlock()
        }

        if !ok {
            return nil, false
        }
        return e.load()
    }
    ```

- 这里有两个值的关注的地方。**一个是首先从 `m.read` 中加载，不存在的情况下，并且 `m.dirty` 中有新数据，加锁，然后从 `m.dirty` 中加载**。

- 二是这里使用了双检查的处理，因为在下面的两个语句中，这两行语句并不是一个原子操作。

    ```go
    if !ok && read.amended {
		m.mu.Lock()
    ```

- 虽然第一句执行的时候条件满足，**但是在加锁之前，`m.dirty` 可能被提升为 `m.read`，所以加锁后还得再检查 `m.read`**，后续的方法中都使用了这个方法。

- 双检查的技术 Java 程序员非常熟悉了，**单例模式的实现之一就是利用双检查的技术**。

- 可以看到

    - **如果我们查询的键值正好存在于 `m.read` 中，`无需加锁`，直接返回，理论上性能优异**。

    - **即使不存在于 `m.read` 中，经过 `miss` 几次之后，`m.dirty` 会被提升为 `m.read`，又会从 `m.read` 中查找**。

    - **所以对于`更新／增加`较少，加载存在的 key 很多的 case，性能基本和`无锁的 map` 类似**。

<br>

- 下面看看 `m.dirty` 是如何被提升的。`missLocked` 方法中可能会将 `m.dirty` 提升。

    ```go
    func (m *Map) missLocked() {
        m.misses++

        if m.misses < len(m.dirty) {
            return
        }

        m.read.Store(readOnly{m: m.dirty})
        m.dirty = nil
        m.misses = 0
    }
    ```

- 上面的最后三行代码就是提升 `m.dirty` 的

    - **很简单的将 `m.dirty` 作为 `readOnly` 的 `m` 字段，原子更新 `m.read`**。
    
    - 提升后 `m.dirty`、`m.misses` 重置，并且 `m.read.amended` 为 `false`。

<br>

## **3.2. Store**
- 这个方法是更新或者新增一个 `entry`。

    ```go
    func (m *Map) Store(key, value interface{}) {
        // 如果 m.read 存在这个键，并且这个 entry 没有被标记删除，尝试直接存储。
        // 因为 m.dirty 也指向这个 entry，所以 m.dirty 也保持最新的 entry。
        read, _ := m.read.Load().(readOnly)
        if e, ok := read.m[key]; ok && e.tryStore(&value) {
            return
        }

        // 如果 `m.read` 不存在或者已经被标记删除
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        if e, ok := read.m[key]; ok {
            if e.unexpungeLocked() { // 标记成未被删除
                m.dirty[key] = e // m.dirty 中不存在这个键，所以加入 m.dirty
            }

            e.storeLocked(&value) // 更新
        } else if e, ok := m.dirty[key]; ok { // m.dirty 存在这个键，更新
            e.storeLocked(&value)
        } else { //新键值
            if !read.amended { // m.dirty 中没有新的数据，往 m.dirty 中增加第一个新键
                m.dirtyLocked() // 从 m.read 中复制未删除的数据
                m.read.Store(readOnly{m: read.m, amended: true})
            }

            m.dirty[key] = newEntry(value) // 将这个 entry 加入到 m.dirty 中
        }
        m.mu.Unlock()
    }

    func (m *Map) dirtyLocked() {
        if m.dirty != nil {
            return
        }

        read, _ := m.read.Load().(readOnly)
        m.dirty = make(map[interface{}]*entry, len(read.m))

        for k, e := range read.m {
            if !e.tryExpungeLocked() {
                m.dirty[k] = e
            }
        }
    }

    func (e *entry) tryExpungeLocked() (isExpunged bool) {
        p := atomic.LoadPointer(&e.p)
        for p == nil {
            // 将已经删除标记为 nil 的数据标记为 expunged
            if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
                return true
            }

            p = atomic.LoadPointer(&e.p)
        }

        return p == expunged
    }
    ```

- 你可以看到，以上操作都是先从操作 `m.read` 开始的，不满足条件再加锁，然后操作 `m.dirty`。

- Store 可能会在某种情况下 (初始化或者 `m.dirty` 刚被提升后) 从 `m.read` 中复制数据，如果这个时候 `m.read` 中数据量非常大，可能会影响性能。

<br>

## **3.3. Delete**
- 删除一个键值。

    ```go
    func (m *Map) Delete(key interface{}) {
        read, _ := m.read.Load().(readOnly)
        e, ok := read.m[key]

        if !ok && read.amended {
            m.mu.Lock()
            read, _ = m.read.Load().(readOnly)
            e, ok = read.m[key]

            if !ok && read.amended {
                delete(m.dirty, key)
            }

            m.mu.Unlock()
        }

        if ok {
            e.delete()
        }
    }
    ```

- 同样，删除操作还是从 `m.read` 中开始，**如果这个 `entry` 不存在于 `m.read` 中，并且 `m.dirty` 中有新数据，则加锁尝试从 `m.dirty` 中删除**。

- 注意，还是要双检查的。从 `m.dirty` 中直接删除即可，就当它没存在过，**但是如果是从 `m.read` 中删除，并不会直接删除，而是`打标记`**：

    ```go
    func (e *entry) delete() (hadValue bool) {
        for {
            p := atomic.LoadPointer(&e.p)

            // 已标记为删除
            if p == nil || p == expunged {
                return false
            }

            // 原子操作，e.p 标记为 nil
            if atomic.CompareAndSwapPointer(&e.p, p, nil) {
                return true
            }
        }
    }
    ```

<br>

## **3.4. Range**
- 因为 `for ... range map` 是内建的语言特性，**所以没有办法使用 `for range` 遍历 `sync.Map`, 但是可以使用它的 `Range` 方法，通过回调的方式遍历**。

    ```go
    func (m *Map) Range(f func(key, value interface{}) bool) {
        read, _ := m.read.Load().(readOnly)

        // 如果 m.dirty 中有新数据，则提升 m.dirty，然后再遍历
        if read.amended {
            // 提升 m.dirty
            m.mu.Lock()
            read, _ = m.read.Load().(readOnly) // 双检查

            if read.amended {
                read = readOnly{m: m.dirty}

                m.read.Store(read)
                m.dirty = nil
                m.misses = 0
            }

            m.mu.Unlock()
        }

        // 遍历, for range 是安全的
        for k, e := range read.m {
            v, ok := e.load()
            if !ok {
                continue
            }

            if !f(k, v) {
                break
            }
        }
    }
    ```

- **Range 方法调用前可能会做一个 `m.dirty` 的提升**，不过提升 `m.dirty` 不是一个耗时的操作。

<br>

## **3.5. sync.Map 的性能**
- Go 1.9 源代码中提供了性能的测试：`map_bench_test.go`、`map_reference_test.go`