# cache2 (archived)

**This repository is archived and unmaintained. It is kept for history only.**

An LRU cache in Go, written in 2012 for [uniqush-push][uniqush-push] and last
changed in 2017. uniqush-push was its only user, and no longer imports it: the
one caller lived in the APNs binary protocol code, which was removed when Apple
switched that protocol off.

**`Delete` is broken.** If you were about to vendor or fork this, read the next
section first.

## The bug

Neither `Cache.Delete` nor `SimpleCache.Delete` removes the element from the LRU
list. They delete the map entry and leave `c.list` holding a `*cacheItem` that
still carries the key:

```go
func (c *SimpleCache) Delete(key string) interface{} {
	c.mu.Lock()
	defer c.mu.Unlock()

	if elem, ok := c.data[key]; ok {
		delete(c.data, key)          // ... but not c.list.Remove(elem)
		item := elem.Value.(*cacheItem)
		return item.value
	}
	return nil
}
```

Three consequences, all measured against `v0.0.1`:

**1. The capacity stops being enforced.** Eviction pops `c.list.Back()` and
deletes that element's key from the map. If the back element is a stale one, the
delete is a no-op, the map does not shrink, and the cache keeps growing.

```go
c := cache.NewSimple(2)
c.Set("a", 1)
c.Set("b", 2)
c.Delete("a")
c.Set("c", 3)
c.Set("d", 4)
c.Len() // 3, with a capacity of 2
```

**2. A live entry can be dropped.** A `Set` after a `Delete` of the same key
pushes a second element for that key, because the map lookup misses. Evicting
the older, stale one runs `delete(c.data, key)` and takes the live value with it:

```go
d := cache.NewSimple(2)
d.Set("a", 1)
d.Delete("a")
d.Set("a", 9) // live again
d.Set("b", 2)
d.Set("c", 3) // evicts the stale "a", which deletes the live "a"
d.Get("a")    // nil, not 9
```

**3. It leaks, invisibly.** The stale elements are unreachable through the map
but still held by the list, and `Len()` reports `len(c.data)`, so the only
instrumentation the type offers cannot see them:

```
200k Set+Delete pairs into a capacity of 256: Len() = 0, heap grew 22 MiB
then 1000 live Sets into that same cache:     Len() = 1000
```

`Set`-only use is unaffected: without a `Delete` there are no stale elements, so
eviction works and the cache stays bounded. uniqush-push happened to be in that
position — its `Delete` calls came from the APNs feedback service, which Apple
had already retired — which is why this went unnoticed for nine years.

The test suite passes, including `TestDeleteValue`. It uses a capacity of 5 with
4 keys, so it never evicts after a delete.

Also worth knowing before you reach for this: `Len()` reads the map without
holding the mutex, so it races every other method, and `CacheInterface` includes
an unexported `debug()`, so no type outside this package can implement it.

## If you want an LRU cache in Go

Use something maintained. [`hashicorp/golang-lru/v2`][golang-lru] is generic,
widely used, and has the eviction semantics this was reaching for. (It is
MPL-2.0, which matters to some projects; this repository is Apache-2.0.)

The bug above is a two-line fix — `c.list.Remove(elem)` in both `Delete`
methods, plus a lock around `Len` — if you would rather fork something small
than take a dependency. Please do that in your own copy; nothing will be merged
here.

License: Apache-2.0

[uniqush-push]: https://github.com/uniqush/uniqush-push
[golang-lru]: https://github.com/hashicorp/golang-lru
