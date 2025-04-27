Java에서 자주 사용하는 HashMap 자료구조가 데이터를 삽입할 때 어떻게 해시 충돌을 해결하는 지 알아보자.

HashMap class의 static 변수들이다.

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 기본 해시 테이블 크기(Bucket 수), 2의 제곱수여야 한다.
static final int MAXIMUM_CAPACITY = 1 << 30; // 해시 테이블 최대 크기
static final float DEFAULT_LOAD_FACTOR = 0.75f; // capacity의 0.75배 만큼 테이블이 차면 해시 테이블 크기를 2배로 늘린다.
static final int TREEIFY_THRESHOLD = 8; // 한 버킷의 연결리스트에 노드가 8개 이상이 되면 RB 트리로 자료구조를 바꿔 성능을 높인다.
static final int UNTREEIFY_THRESHOLD = 6; // RB 트리 였던 버킷에 노드가 6개 이하가 되면 다시 연결 리스트로 바꾼다.
static final int MIN_TREEIFY_CAPACITY = 64; // 한 버킷의 노드가 8개 이상이더라도 전체 크기가 64 이상 이어야 트리로 바꾼다.
```

static class로 정의되어 있는 Node를 살펴보자.
```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;

        return o instanceof Map.Entry<?, ?> e
                && Objects.equals(key, e.getKey())
                && Objects.equals(value, e.getValue());
    }
}
```
next 변수를 보면 알겠지만 기본적으로 버킷은 연결 리스트로 이루어져 있다. 즉 해시 충돌이 발생하면 **체이닝 방식**으로 해결한다.

해시 테이블에 데이터를 어떻게 추가하는 지 살펴보자.
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
우리가 개발할 때는 put 메소드를 사용하지만 내부적으로는 putVal 메소드를 호출한다.

1. 아직 해시 테이블이 없으면 resize()를 통해 해시 테이블을 초기화한다.
```java
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
```
2. 버킷 배열의 인덱스(i)에 노드가 없으면 새로운 노드를 추가한다.
```java
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```
3. 버킷 배열의 인덱스(i)에 노드가 있을 때, 버킷의 헤드 노드와 내가 넣으려는 데이터가 같은 해시 값, 키를 갖고 있는지 확인한다.
```java
Node<K,V> e; K k;
  if (p.hash == hash &&
      ((k = p.key) == key || (key != null && key.equals(k))))
      e = p;
```
4. 만약 해당 버킷이 연결리스트가 아닌 RB 트리 구조라면, 트리에 저장한다.
```java
else if (p instanceof TreeNode)
    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```
5. 그냥 연결리스트라면 연결리스트를 확인하면서 같은 키를 찾고, 없으면 새 노드를 추가한다. 이 때, 버킷 크기가 일정 크기를 초과하면 RB 트리 구조로 변경한다.
```java
else {
    for (int binCount = 0; ; ++binCount) {
        if ((e = p.next) == null) {
            p.next = newNode(hash, key, value, null);
            if (binCount >= TREEIFY_THRESHOLD - 1)
                treeifyBin(tab, hash);
            break;
        }
        if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
            break;
        p = e;
    }
}
```
6. 버킷에 같은 키가 있다면 값을 덮어쓴다. afterNodeAccess는 LinkedHashMap에서 순서 갱신을 위해 호출된다.
```java
if (e != null) {
    V oldValue = e.value;
    if (!onlyIfAbsent || oldValue == null)
        e.value = value;
    afterNodeAccess(e);
    return oldValue;
}
```
7. 같은 키가 없다면 새로운 노드를 추가 후 테이블 크기를 확인해 필요하면 resize 한다. afterNodeInsertion은 LinkedHashMap에서 순서 갱신을 위해 호출된다.
```java
++modCount;
if (++size > threshold)
    resize();
afterNodeInsertion(evict);
return null;
```

## 결론
Java의 HashMap은 기본적으로 각 버킷을 연결 리스트 구조로 두어 해시 충돌이 발생했을 때 체이닝 방식으로 해결한다. 그러다가 버킷(연결리스트)의 크기가 일정 크기 이상이 되면 성능 개선을 위해 버킷을 RB 트리 구조로 변경한다.
RB 트리는 균형잡힌 트리 종류 중 하나로 삽입,삭제,검색의 시간복잡도가 O(logN)이기 때문이다.
그러나 이 HashMap은 putVal 메소드를 보다시피 Thread-Safe하지 않다. 같은 버킷에 대해 잘못된 크기 계산이나 잘못된 트리 구조 변환이 발생할 수 있다. 따라서 CAS와 락을 이용해 멀티 스레드에서도 안전한 ConcurrentHashMap도 있다.







