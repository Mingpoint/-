### HashMap 为什么线程不安全
#### 1.jdk1.7中的HashMap
- 1.1 扩容造成死循环分析过程
- 1.2 扩容造成数据丢失分析过程

#### jdk1.7中的HashMap
- 2.1 数据覆盖，发生线程不安全

#### 1.7 HashMap 链表使用头插法
#### 1.8 HashMap 链表使用尾插法

#### 1.7 ConcurrentHashMap
```java
    //Segment继承了ReentrantLock
    static final class Segment<K,V> extends ReentrantLock implements Serializable
```

put方法
```java
    final V put(K key, int hash, V value, boolean onlyIfAbsent) {
                // 加锁
                HashEntry<K,V> node = tryLock() ? null :
                    scanAndLockForPut(key, hash, value);
    }
```
scanAndLockForPut方法
```java
       private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
            HashEntry<K,V> first = entryForHash(this, hash);
            HashEntry<K,V> e = first;
            HashEntry<K,V> node = null;
            int retries = -1; // negative while locating node
            //自旋加锁
            while (!tryLock()) {
                HashEntry<K,V> f; // to recheck first below
                if (retries < 0) {
                    if (e == null) {
                        if (node == null) // speculatively create node
                            node = new HashEntry<K,V>(hash, key, value, null);
                        retries = 0;
                    }
                    else if (key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
                 //如果尝试加锁次数超过最大，直接加锁并返回对象，最大值是 核心数> 1 ? 64 : 1;       
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
                else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                    e = first = f; // re-traverse if entry changed
                    retries = -1;
                }
            }
            return node;
        }
```
#### 总结：1.7ConcurrentHashMap 通过Segment加锁，Segment继承了ReentrantLock实现加锁过程