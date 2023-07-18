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
    
    // 如果 p == nil，则表示该 entry 已被删除，且 m.dirty == nil 或 m.dirty[key] == e
    // 如果 p == expunged，则表示该 entry 已被删除，m.dirty != nil，且该 entry 在 m.dirty 中不存在
    //否则，该 entry 是有效的，并记录在 m.read.m[key] 中，在 m.dirty != nil 的情况下，也记录在 m.dirty[key] 中。
    // 可以通过原子替换为 nil 来删除 entry，当 m.dirty 被创建时，它将原子替换为 expunged 并将 m.dirty[key] 置为空。可以通过原子替换来更新 entry 的关联值，前提是 p != expunged。如果 p == expunged，则只有在首先设置 m.dirty[key] = e 以便使用 dirty 映射表查找该 entry 后，才能更新 entry 的关联值。
	p atomic.Pointer[any]
}
```

