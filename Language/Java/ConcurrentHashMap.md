Java에서 일반적인 HashMap은 멀티스레드 환경에서 Thread-Safe하지 않기 때문에 동시성 문제가 발생할 수 있다. Java에선 Thread-Safe한 HashMap을 제공하는데 이것이 ConcurrentHashMap이다.

ConcurrentHashMap은 내부적으로 분할 잠금 메커니즘(Lock Stripping)과 CAS(Compare-And-Swap) 같은 비동기적인 동시성 제어 기법을 사용하여 여러 스레드가 동시에 데이터를 읽고 쓰는 상황에서도 안전하게 동작한다.

## ConcurrentHashMap이란?

- 자바의 동시성 컬렉션 클래스 중 하나로, 멀티 스레드 환경에서 안전하게 사용할 수 있다.

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    // ... code
}
```

- ConcurrentHashMap에서 읽기 작업은 별다른 동기화 작업 없이 빠르게 수행되나 쓰기 작업에서는 여러 스레드가 동시에 같은 키에 접근하거나 데이터를 수정할 때 동시성 문제가 발생할 수 있다.
- 쓰기 작업을 처리하는 메소드를 살펴보자.

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

public V putIfAbsent(K key, V value) {
    return putVal(key, value, true);
}
```

- 위와 같이 put과 putIfAbsent 메소드의 내부에서는 putVal 메소드를 호출한다.

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // ...
}
```

- putVal은 private 메소드로 onlyIfAbsent 인자는 키가 없을 때만 값을 삽입할 지 키가 있더라도 무조건 삽입할 지 결정한다.
- 따라서 put 메소드를 사용하면 기존 값이 업데이트되며 putIfAbsent 메소드를 사용하면 기존 값이 없을 때만 값을 삽입한다.

## CAS(Compare-And-Swap)란 무엇인가?

- CAS는 비동기적 동시성 제어 기법 중 하나로 여러 스레드가 동시에 데이터를 수정하려고 할 때 데이터의 일관성을 보장하는 방법이다. CAS는 원자적(atomic) 연산이기 때문에 안전하게 데이터를 읽고 쓸 수 있다.
- CAS는 메모리에 저장된 기존 값과 각 스레드의 스택에 저장된 값을 비교하여 같으면 수정하는 방법이다.

## CAS 연산은 HW 수준에서 원자적으로 수행된다

- 현대 CPU는 CAS 연산을 지원하는 명령어들이 있는데 CAS 연산은 원자적 연산이기 때문에 연산이 실행되는 동안 다른 어떤 연산(다른 스레드나 프로세스)도 해당 메모리 주소에 접근할 수 없다. 명령어가 끊기지 않고 한 번에 수행되기 때문에 메모리 주소를 읽고, 비교하고, 값을 교체하는 과정이 한 번에 이루어진다. 따라서 CAS 연산은 완전히 수행되거나 전혀 수행되지 않는 2가지 상태만을 가진다.

## CAS가 중요한 이유

- Lock을 사용하지 않아 DeadLock이 발생하지 않는다.
- 높은 성능 및 비동기적 처리

## ConcurrentHashMap에선 CAS 기법을 어떻게 사용하고 있을까?

- putVal 메소드의 일부를 확인해보자.

```java
else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
    // 여기서 사용된다.
    if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
        break; // 락 없이 빈 버킷에 노드를 추가한 경우 반복을 종료합니다.
}
```

- tabAt(tab, i)는 tab이라는 내부 배열의 i번째 인덱스에 있는 버킷을 확인한다. 이 때 버킷이 비어있다면 (null 이라면) CAS 연산인 casTabAt을 호출해 해당 위치가 여전히 null일 때만 새로운 노드를 원자적으로 삽입한다.

```java
@SuppressWarnings("unchecked")
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getReferenceAcquire(tab, ((long)i << ASHIFT) + ABASE);
}

@IntrinsicCandidate
public final Object getReferenceAcquire(Object o, long offset) {
    return getReferenceVolatile(o, offset);
}

@IntrinsicCandidate
public native Object getReferenceVolatile(Object o, long offset);

```

- tabAt 메소드 코드를 보면 상당히 복잡한데 자바 네이티브 인터페이스(JNI)를 통해 메모리에 안전하게 접근한다.

```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSetReference(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

@IntrinsicCandidate
public final native boolean compareAndSetReference(Object o, long offset,
                                                   Object expected,
                                                   Object x);
```

- casTabAt 메소드 내부에서는 tab[i]의 값이 여전히 Null 이면 새로운 노드를 삽입하고 다른 스레드가 노드를 삽입했다면 삽입하지 않는다.

**→ 쉽게 생각하면 자리가 비어 있다면 내 물건을 두고 다른 사람이 먼저 물건을 두어 자리가 비어 있지 않다면 물건을 두지 않는 것이다.**

## ConcurrentHashMap의 putVal 메소드 분석

```java
if (key == null || value == null) throw new NullPointerException();
int hash = spread(key.hashCode());
int binCount = 0;
```

- 기본적으로 key나 value가 null 값이면 NullPointerException이 터진다.
- spread 메소드는 해시 값을 적절하게 분산시켜 해시 충돌을 줄여 성능을 최적화 시킨다.

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
}
```

- 참고로 tab 테이블은 Node<K, V>[] 배열이다. static class로 선언되어 있다.

```java
for (Node<K,V>[] tab = table;;) {
		Node<K,V> f; int n, i, fh; K fk; V fv;
		
    if (tab == null || (n = tab.length) == 0)
        tab = initTable();
                
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
        if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
            break; // 락 없이 빈 버킷에 노드를 추가한 경우 반복을 종료합니다.
    }
}
```

- tab 배열이 비어있다면 테이블을 초기화한다.
- i번째 버킷이 비어있다면 casTabAt 메소드를 통해 i번째 인덱스 버킷에 노드를 추가한다. 만약 CAS 연산이 성공해 노드가 안전하게 추가 되었다면 break 문을 통해 반복문을 탈출한다.

```java
for (Node<K,V>[] tab = table;;) {
    ...
    
    else if ((fh = f.hash) == MOVED) {
        tab = helpTransfer(tab, f);
        ...
    }
    
    else if (onlyIfAbsent 
               && fh == hash
               && ((fk = f.key) == key || (fk != null && key.equals(fk)))
               && (fv = f.val) != null) {
        return fv;
    }
}
```

- f.hash가 MOVED 값이라면 해시 테이블이 특정 임계치에 도달하여 크기를 늘려야 한다.
- 버킷의 첫 번째 노드만 빠르게 확인해서 키 값이 이미 있다면 기존 값을 반환하고 빠르게 끝낸다.

```java
for (Node<K,V>[] tab = table;;) {
		
		...
				
    else {
        V oldVal = null;
        synchronized (f) {
            if (tabAt(tab, i) == f) {
		            // LinkedList 구조일 때
                if (fh >= 0) {
                    binCount = 1;
                    for (Node<K,V> e = f;; ++binCount) {
                        K ek;
                        if (e.hash == hash &&
                            ((ek = e.key) == key ||
                             (ek != null && key.equals(ek)))) {
                            oldVal = e.val;
                            if (!onlyIfAbsent)
                                e.val = value;
                            break;
                        }
                        Node<K,V> pred = e;
                        if ((e = e.next) == null) {
                            pred.next = new Node<K,V>(hash, key, value);
                            break;
                        }
                    }
                }
                // RedBlack Tree 구조일 때 (TreeBin 구조일 때)
                else if (f instanceof TreeBin) {
                    Node<K,V> p;
                    binCount = 2;
                    if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                        oldVal = p.val;
                        if (!onlyIfAbsent)
                            p.val = value;
                    }
                }
                else if (f instanceof ReservationNode)
                    throw new IllegalStateException("Recursive update");
            }
        }
        ...
    }
}
```

- f는 i번째 버킷의 첫 번째 노드이다.
- 마지막 else 구문은 i번째 버킷은 있지만 버킷 속에 key 값이 없어서 값을 삽입하는 경우다. 이 경우에는 안전하게 synchronized 구문을 통해 락을 걸어 해당 버킷 내부에 안전하게 값을 추가한다.
- 여기서 LinkedList와 Tree 구조로 분기가 나뉜 이유는 하나의 버킷에 노드가 너무 많아지면 (기본 8개 이상) 성능 저하 문제로 인해 자동으로 Red Black Tree 구조로 바뀌기 때문이다.

```java
for (Node<K,V>[] tab = table;;) {
    ...
    
    if (binCount != 0) {
		    if (binCount >= TREEIFY_THRESHOLD)
		        treeifyBin(tab, i);
		    if (oldVal != null)
		        return oldVal;
		    break;
		}
	  addCount(1L, binCount);
    return null;
}
```

- 이 코드가 버킷의 노드 숫자가 기본값인 8개 이상이 되면 Red Black Tree 구조 (TreeBin 구조)로 바꾸는 부분이다.
- addCount는 해시 테이블의 전체 노드 숫자를 증가시키고 특정 조건에서 테이블을 리사이징한다.

## 결론

- 버킷이 비어있으면 CAS로 경쟁 없이 바로 삽입하는데 락을 걸지 않아 빠르다.
- 버킷이 존재하고 key도 있고 onlyIfAbsent(putIfAbsent)인 경우에는 락 없이 기존 값을 반환한다.
- 버킷이 존재하는데 key가 없거나 put인 경우에는 synchronized로 버킷에 락을 걸고 스레드가 하나씩 안전하게 처리된다.
- 따라서 멀티 스레드 환경에서도 안전하게 값이 저장된다.
