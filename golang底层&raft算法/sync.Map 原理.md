#### Q1. 简单使用用例

```go
func Test_sync_map(t *testing.T) {
	var smap sync.Map
	// 存储
	smap.Store("name", "zhangsan")
	smap.Store("age", 18)
	smap.Store("dsd", "dassas")
	// 读取
	name, ok := smap.Load("name")
	if !ok {
		t.Log("key does not exist")
		return
	}
	t.Log(name)
	//	删除
	smap.Delete("dsd")
	//dsd, ok := smap.Load("dsd")
	//if !ok {
	//	t.Log("key does not exist")
	//	return
	//}
	//t.Log(dsd)

	// 遍历
	smap.Range(func(key, value interface{}) bool {
		t.Log("k: ", key, " v: ", value)
		return true
	})

}
```

#### Q2. 数据结构

![image-20230719002704990](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230719002704990.png)

以空间换时间的策略冗余出一个map

一个map只读(无锁化),另外一个map可读可写(加锁)

```go
type Map struct {
    //一个互斥锁，用于保护 Map 的并发访问
	mu Mutex

	//read 包含可以安全并发访问的映射内容部分（无论是否持有 mu）。
     //读取字段本身始终可以安全加载，但只能使用 mu 进行存储。存储在 read 中的条目可以在没有 mu 的情况下同时更新，但是更新先前删除的条目需要将该条目复制到脏映射并在保持 mu 的情况下取消删除
    //一个 atomic.Pointer 类型的指针，指向一个 readOnly 类型的结构体，其中包含了 Map 的部分内容，可以在不持有 mu 的情况下并发访问
	read atomic.Pointer[readOnly]

	// dirty contains the portion of the map's contents that require mu to be
	// held. To ensure that the dirty map can be promoted to the read map quickly,
	// it also includes all of the non-expunged entries in the read map.
	//
	// Expunged entries are not stored in the dirty map. An expunged entry in the
	// clean map must be unexpunged and added to the dirty map before a new value
	// can be stored to it.
	//
	// If the dirty map is nil, the next write to the map will initialize it by
	// making a shallow copy of the clean map, omitting stale entries.
    // 一个 map 类型的映射表，包含了需要持有 mu 才能访问的 Map 的内容。为了快速将 dirty 映射表提升为 read 映射表，dirty 映射表还包含了 read 映射表中所有未被删除的条目
	dirty map[any]*entry

	// misses counts the number of loads since the read map was last updated that
	// needed to lock mu to determine whether the key was present.
	//
	// Once enough misses have occurred to cover the cost of copying the dirty
	// map, the dirty map will be promoted to the read map (in the unamended
	// state) and the next store to the map will make a new dirty copy.
    // 一个整数类型的计数器，用于记录自上次更新 read 映射表以来需要锁定 mu 才能确定键是否存在的加载次数。
    // 当 misses 的值足够大时，dirty 映射表将被提升为 read 映射表，并且下一次对 Map 的写操作将创建一个新的 dirty 映射表
    // 标识只读map与可读写map的差异性
	misses int
}


type readOnly struct {
    // 一个 map 类型的映射表，包含了 Map 的部分内容，可以在不持有 mu 的情况下并发访问
	m       map[any]*entry 
    // 一个布尔类型的标志位，表示 dirty 映射表是否包含了 m 映射表中不存在的键。如果为 true，则表示 dirty 映射表包含了一些 m 映射表中不存在的键
	amended bool // true if the dirty map contains some key not in m.
}





// An entry is a slot in the map corresponding to a particular key.
type entry struct {
	// p points to the interface{} value stored for the entry.
	//
	// If p == nil, the entry has been deleted, and either m.dirty == nil or
	// m.dirty[key] is e.
	//
	// If p == expunged, the entry has been deleted, m.dirty != nil, and the entry
	// is missing from m.dirty.
	//
	// Otherwise, the entry is valid and recorded in m.read.m[key] and, if m.dirty
	// != nil, in m.dirty[key].
	//
	// An entry can be deleted by atomic replacement with nil: when m.dirty is
	// next created, it will atomically replace nil with expunged and leave
	// m.dirty[key] unset.
	//
	// An entry's associated value can be updated by atomic replacement, provided
	// p != expunged. If p == expunged, an entry's associated value can be updated
	// only after first setting m.dirty[key] = e so that lookups using the dirty
	// map find the entry.
    // entry.p 的指向分三种情况
    // 1. 存活态: p正常指向元素;
    // 2. 软删除: p指向nil;(逻辑上删除)
    // 3. 硬删除态: p指向固定的全局变量 expunged
	p atomic.Pointer[any]
}
```

结构体"entry"包含以下字段：

- "p"：指向存储在该键值对中的接口值（interface{} value）的指针。
  - 如果"p"为nil，表示该键值对已被删除，此时可能有两种情况：
    - 若"m.dirty"为nil，则该键值对被删除，且映射中的"m.dirty"为空或"m.dirty[key]"为当前键值对。
    - 若"m.dirty"不为nil，则该键值对被删除，且映射中的"m.dirty"不包含该键值对。
  - 如果"p"为"expunged"，表示该键值对已被删除，"m.dirty"不为nil，且映射中的"m.dirty"不包含该键值对。
  - 否则，表示该键值对是有效的，并记录在"m.read.m[key]"中，如果"m.dirty"不为nil，还会记录在"m.dirty[key]"中。

通过原子替换操作，可以将一个键值对从映射中删除，将其"p"字段替换为nil。当下次创建"m.dirty"时，它会原子性地将nil替换为"expunged"，并保持"m.dirty[key]"未设置。

通过原子替换操作，可以更新一个键值对的关联值，前提是"p"不等于"expunged"。如果"p"等于"expunged"，则只有在首先设置"m.dirty[key] = e"以便使用脏映射（dirty map）进行查找时，才能更新键值对的关联值。

#### Q3. 读流程

![image-20230719003011128](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230719003011128.png)

```go
func (m *Map) Load(key any) (value any, ok bool) {
    //调用 loadReadOnly() 方法获取一个只读的副本 read，该副本包含当前的映射数据和标志位。
	read := m.loadReadOnly()
    //从 read 中查找给定 key 对应的值 e。如果存在，则将 ok 设置为 true，否则为 false。
	e, ok := read.m[key]
    // 如果在 read 中没有找到该 key 并且标志位 amended 为真(当前只读map数据缺失)，则需要考虑从 dirty 映射中查找
	if !ok && read.amended {
		m.mu.Lock()
		// Avoid reporting a spurious miss if m.dirty got promoted while we were
		// blocked on m.mu. (If further loads of the same key will not miss, it's
		// not worth copying the dirty map for this key.)
        // double cherk 再次获取只读的副本 read，此时包含了最新的映射数据和标志位。
		read = m.loadReadOnly()
		e, ok = read.m[key]
        // 如果在新的 read 中仍然没有找到该 key 并且标志位 amended 为真，则从 dirty 映射中查找
		if !ok && read.amended {
            //从 dirty 映射中查找给定 key 对应的值 e。如果找到，则将 ok 设置为 true。
			e, ok = m.dirty[key]
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
            // 记录一次未命中操作，无论该键是否存在。这样可以确保该键会在之后的慢速路径中处理，直到 dirty 映射被提升为读取映射。 miss++
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}

func (m *Map) loadReadOnly() readOnly {
     
	if p := m.read.Load(); p != nil {
		return *p
	}
	return readOnly{}
}

func (e *entry) load() (value any, ok bool) {
   
	p := e.p.Load()
    // 检查 p 的值，如果它为 nil 或者等于 expunged（一个特殊的标记），则表示该值已被删除或未设置。
	if p == nil || p == expunged {
		return nil, false
	}
    //如果值存在且有效，则返回解引用后的值 *p 和 true，表示成功加载值。 
	return *p, true
}

// Load atomically loads and returns the value stored in x.
// Load 以原子方式加载并返回存储在 x 中的值。
func (x *Pointer[T]) Load() *T { return (*T)(LoadPointer(&x.v)) }


func (m *Map) missLocked() {
	m.misses++
    // 如果未命中计数器小于 dirty 映射的长度，表示还未达到触发条件，直接返回，不执行后续操作
	if m.misses < len(m.dirty) {
		return
	}
    // 将 dirty 映射转换为只读的 readOnly 结构体，并使用 Store 方法将其存储到 read 中。readOnly 结构体是映射的只读副本，用于并发安全的读取操作。
	m.read.Store(&readOnly{m: m.dirty})
    // 将 dirty 映射置为空，表示所有的写入操作都已经被转移到了 read 中。
	m.dirty = nil
    // m.misses = 0：将未命中计数器重置为 0，以便开始下一个计数周期
	m.misses = 0
}

```

#### Q4. 写流程

![image-20230719004546765](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230719004546765.png)

```GO
// Store sets the value for a key.
func (m *Map) Store(key, value any) {
	_, _ = m.Swap(key, value)
}

//Swap 方法用于交换给定键的值，并返回交换前的旧值
// 它首先在只读的 read 映射中查找给定的 key
	//如果找到并成功交换值，则返回交换前的旧值和 true
	//如果未找到该键或交换失败，则获取互斥锁，并重新检查 read 和 dirty 映射
	//如果在其中任何一个映射中找到该键，并成功交换值，则返回交换前的旧值和 true
	// 如果在两个映射中均未找到该键，则创建一个新的 dirty 映射，并将新的值添加到其中
	// 最后，释放互斥锁，并返回旧值和加载状态

// Swap swaps the value for a key and returns the previous value if any.
// The loaded result reports whether the key was present.
func (m *Map) Swap(key, value any) (previous any, loaded bool) {
    // 利用ReadOnly 应对更新操作
	read := m.loadReadOnly()
	if e, ok := read.m[key]; ok {
        // 调用 trySwap 方法尝试原子交换值，并返回新值 v 和是否成功交换的标志位。
		if v, ok := e.trySwap(&value); ok {
            // 如果新值 v 为 nil，表示交换失败，返回空值和 false。
			if v == nil {
				return nil, false
			}
			return *v, true
		}
	}

	m.mu.Lock()
    // 重新获取只读的副本 read，以确保操作最新的映射数据。
	read = m.loadReadOnly()
    // 再次检查给定的 key 是否存在于新的 read 中。
	if e, ok := read.m[key]; ok {
        // 存在在read中
        // 如果 e 已被标记为删除（expunged），则调用 unexpungeLocked 方法将其恢复，并返回 true。
		if e.unexpungeLocked() {
			// The entry was previously expunged, which implies that there is a
			// non-nil dirty map and this entry is not in it.
            // 将 e 添加到 dirty 映射中，表示该键已从 read 中恢复
			m.dirty[key] = e
		}
        // 调用 swapLocked 方法原子交换值，并返回交换前的旧值 v
		if v := e.swapLocked(&value); v != nil {
            //设置 loaded 为 true 表示成功加载旧值
			loaded = true
            //将旧值 *v 赋给 previous
			previous = *v
		}
        // 如果在 read 中未找到该键，并且该键存在于 dirty 映射中，则执行后续逻辑
	} else if e, ok := m.dirty[key]; ok {
        // 调用 swapLocked 方法原子交换值，并返回交换前的旧值 v。如果交换成功，执行后续逻辑。
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
        //如果在 read 和 dirty 中均未找到该键
        // 说明是新增而不是更新
	} else {
        // 如果 read 的标志位 amended 为假，表示 dirty 映射为空/readonly 全量相等于dirty映射，需要创建一个新的 dirty 映射
		if !read.amended {
			// We're adding the first new key to the dirty map.
			// Make sure it is allocated and mark the read-only map as incomplete.
            // 拷贝map。
			m.dirtyLocked()
            // 将标志位 amended 设置为真，并将新的 read 存储到 Map 中。
			m.read.Store(&readOnly{m: read.m, amended: true})
		}
        // 将新的 entry（包含新值）添加到 dirty 映射中。
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
    //返回旧值 previous 和加载状态 loaded
	return previous, loaded
}

// trySwap swaps a value if the entry has not been expunged.
//
// If the entry is expunged, trySwap returns false and leaves the entry
// unchanged.
func (e *entry) trySwap(i *any) (*any, bool) {
	for {
		p := e.p.Load()
        // 如果 p 的值等于 expunged，表示该条目已被标记为硬删除
		if p == expunged {
			return nil, false
		}
        // if e.p.CompareAndSwap(p, i)：尝试使用原子比较并交换（CAS）操作将指针 p 替换为新值 i。
		if e.p.CompareAndSwap(p, i) {
			return p, true
		}
	}
}


// unexpungeLocked 可确保条目未标记为删除.
//
// 如果先前已删除该条目，则必须在M.MU解锁之前将其添加到dirty map 中
//
func (e *entry) unexpungeLocked() (wasExpunged bool) {
	return e.p.CompareAndSwap(expunged, nil)
}


func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read := m.loadReadOnly()
	m.dirty = make(map[any]*entry, len(read.m))
    
    // 瓶颈点:
    //   规避使用情景:写(插入)多读少
	for k, e := range read.m {
        // 将软删除状态的数据更新为硬删除状态
        // 只copy确切未删除的数据
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := e.p.Load()
     // 将软删除状态的数据更新为硬删除状态
	for p == nil {
		if e.p.CompareAndSwap(nil, expunged) {
			return true
		}
		p = e.p.Load()
	}
	return p == expunged
}

```

