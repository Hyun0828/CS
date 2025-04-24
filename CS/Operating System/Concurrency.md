# Concurrency

# 목차
- [Concurrency and Threads](#26-concurrency-and-threads)
  - [데이터 공유](#데이터-공유)
  - [원자성](#원자성)
- [Locks](#28-locks)
  - [락](#락)
  - [Pthread](#pthread)
  - [락 구현](#락-구현)
  - [인터럽트 제어](#인터럽트-제어)
  - [Test-And-Set](#test-and-set)
  - [스핀 락 평가](#스핀-락-평가)
  - [Compare-And-Swap](#compare-and-swap)
  - [Fetch-And-Add](#fetch-and-add)
  - [과도한 스핀](#과도한-스핀)
  - [무조건 양보!](#무조건-양보)
  - [잠자기](#잠자기) 

## 26. Concurrency and Threads

쓰레드는 프로세스에서의 하나의 실행 흐름이다. 쓰레드마다 PC와 연산을 위한 레지스터를 가지고 있다. 만약 2개의 쓰레드가 문맥 교환을 하는 상황이라면 실행 중인 쓰레드의 레지스터 내용을 PCB가 아닌 TCB에 저장하고 새로 실행될 쓰레드의 TCB에서 레지스터 값을 복원한다. 마치 프로세스의 문맥교환과 매우 흡사하다. 또한 쓰레드끼리 주소 공간을 공유하지만 스택은 따로 할당된다.

<img width="557" alt="스크린샷 2025-04-22 오후 10 14 35" src="https://github.com/user-attachments/assets/a00b0395-a980-43e4-bd6c-3a6eb89b35e9" />

스레드에서 매개변수, 리턴 값 같은 데이터는 스레드 고유의 스택 공간에 저장된다. 

이런 스레드들은 생성되면 즉시 실행될 수도 있고, 준비 상태에서 실행은 되지 않을 수도 있다. 만약 메인 스레드에서 2개의 스레드를 생성하고 특정 함수를 실행 시키고자 할 때, 스레드의 실행 순서와 함수 실행 순서 등은 정해지지 않고 매번 바뀐다. 

### 데이터 공유

스레드 실행 순서는 스케줄러의 동작에 따라 바뀔 수 있다. 만약 여러 스레드가 공유 데이터에 접근하면 어떻게 될까? 2개의 스레드가 전역 공유 변수를 갱신하는 코드를 살펴보자.

```c
#include <stdio.h>
#include <pthread.h>
#include "mythreads.h"

static volatile int counter = 0;

void *mythread(void *arg){
	// counter 값을 10,000,000번 +1 하는 로직
}

int main(int argc, char *argv[]){
	// 스레드 2개를 생성하고 각 스레드에서 mythread 함수 실행
}
```

우리는 counter 값이 20,000,000이 되기를 기대한다. 그러나 실제로는 이상한 값이 나온다.

counter에 1을 더하는 코드의 어셈블리 코드를 보면 다음과 같다.

```nasm
mov 0x8049a1c, %eax
add $0x1, %eax
mov %eax, 0x8049a1c
```

현재 counter 값이 50이라고 해보자. 스레드1이 mov, add까지 실행하고 인터럽트가 발생하면 스레드1의 레지스터 값들을 TCB에 저장하고 스레드2가 실행된다. 스레드2의 eax 레지스터에는 50이 저장된다. 스레드1에서 마지막 mov 명령어를 실행하지 않았기 때문이다. 스레드2에서 모든 명령어를 실행하고 counter 값이 51이 된 후 스레드1로 돌아온다고 하면 counter 값은 여전히 51이다.

이렇게 명령어의 실행 순서에 따라 결과가 달라지는 상황을 **race condition(경쟁 조건)** 이라고 한다. 멀티 프로세스나 멀티 스레드 환경에서 race condition이 발생하는 코드 부분을 **critical section(임계 구역)** 이라고 한다. 이러한 코드에서 필요한 것은 **mutual exclusion(상호 배제)** 이다. 상호 배제는 임계 구역에 하나의 스레드만 실행할 수 있게 보장해주는 것이다.

### 원자성

상호 배제 말고도 다양한 해결책들이 있다. 원자성을 보장하는 메소드를 이용하면 메소드 실행 중간에 인터럽트가 발생하지 않아 race condition이 발생하지 않는다. 상호 배제를 구현하기 위해서는 보통 동기화 함수들을 사용한다. 다른 방법으로 스레드가 잠들고 다른 스레드를 깨우는 **condition variable** 을 이용한 방법도 있다.

## 28. Locks

### 락

고전적인 공유 변수 갱신 예제를 통해 락의 기본 개념을 확인하자.

```c
lock_t mutex;
...
lock(&mutex);
balance = balance + 1;
unlock(&mutex);
```

lock변수와 lock(), unlock()을 통해 상호 배제를 구현할 수 있다. 락을 획득하고 임계 구역에 진입한 쓰레드를 락 소유자라고 한다. 소유자가 락을 해제하면 대기하던 쓰레드가 락을 획득해 새로 임계 구역에 진입한다. 일반적으로 쓰레드에 대한 스케줄링 제어권은 운영체제가 가지고 있으나 락을 이용하면 프로그래머가 제어권을 일부 이양 받을 수 있다.

### Pthread

상호 배제 기능을 제공하는 mutex 락이다. 

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
...
Pthread_mutex_lock(&lock);
balance = balance + 1;
Pthread_mutex_unlock(&lock);
```

### 락 구현

그럼 이러한 락은 어떻게 구현되어 있을까? 효율적인 락을 만들기 위해서는 하드웨어와 운영체제의 도움을 받으면 된다. **효율적인 락은 어떤 락일까?** 첫째로 **상호 배제** 를 제대로 지원해야 한다. 둘째로는 **공정성** 이다. 락을 대기하는 여러 쓰레드에게 공정한 기회가 주어져야 한다. 마지막 기준은 **성능** 이다. 여러 쓰레드가 단일 CPU에서 경쟁하는 경우, 멀티 CPU에서 경쟁하는 경우 등에 대해 락 사용 시간적 오버헤드가 중요하다. 

### 인터럽트 제어

초기 단일 프로세스 시스템에서는 상호 배제를 지원하기 위해 **임계 영역 내에서는 인터럽트를 비활성화 했다.** 이렇게 되면 임계 영역 내에서는 인터럽트가 발생하지 않아 원자적으로 실행될 수 있다. 그러나 락을 획득한 쓰레드에서 오류가 발생하거나 무한 반복문에 빠져도 인터럽트를 발생 시키지 못해 프로세서를 독점하는 문제가 있다. 또 다른 문제는 멀티프로세서에서 적용할 수 없다는 것이다. CPU0에서의 인터럽트 비활성화가 CPU1에는 영향을 끼치지 않기 때문이다. 세 번째로는 인터럽트가 발생해야 하는 중요한 시점을 놓칠 수 있으며 마지막으로 인터럽트 비활성화 명령 자체가 성능이 굉장히 느리다.

### Test-And-Set

인터럽트 비활성화를 대신하여 락 지원을 위한 하드웨어 설계가 이루어졌다. 하드웨어 기법 중 가장 기본은 **Test-And-Set** 명령어 이다. **원자적 교체** 라고도 불린다.

```c
typedef struct __lock_t {int flag;} lock_t;

void init(lock_t *mutex) {mutex->flag = 0;}

void lock(lock_t *mutex){
	while(mutex->flag == 1) ;
	mutex->flag = 1;
}

void unlock(lock_t *mutex) {mutex->flag = 0;}
```

아이디어는 간단하다. 락을 획득하려 할 때 flag 값이 0이면 락을 획득하고 flag 값을 1로 만든다. flag 값이 1이면 while문으로 **spin-wait** 하며 락을 대기한다. 그러나 이 방식에는 2가지 문제가 있다. 

우선 쓰레드1이 while문을 실행할 때 쓰레드2로 문맥 교환이 발생해 쓰레드2에서 flag 값을 1로 변경한 후 쓰레드1로 돌아오면 쓰레드1도 flag 값을 1로 변경하기 때문에 **상호 배제가 깨진다.**

![image (6)](https://github.com/user-attachments/assets/9415f10c-d86e-458b-a6e2-5f05f1844fdb)

또한 **spin-wait** 방식은 플래그 값을 무한히 검사하는데, 이 방법은 락 소유자가 락을 해제할 때까지 시간을 낭비하며 CPU를 계속 소모하는 문제가 있다.

그래서 원자적 교체 명령어인 TestAndSet을 활용한다.

```c
int TestAndSet(int *old_ptr, int new){
	int old = *old_ptr;
	*old_ptr = new;
	return old;
}
```

TestAndSet은 기존 값을 새로운 값으로 교체하고 기존 값을 반환한다.

```c
typedef struct __lock_t {int flag;} lock_t;

void init(lock_t *lock) {lock->flag = 0;}

void lock(lock_t *lock){
	while(TestAndSet(&lock->flag, 1)==1) ;
}

void unlock(lock_t *lock) {lock->flag = 0;}
```

위와 같이 변경하면 기존의 상호 배제가 깨지는 문제를 해결할 수 있다. **TestAndSet이 원자적 연산이기 때문이다.** 단일 프로세서에서 이 방식을 제대로 사용하려면 선점형 스케줄러를 사용해야 한다. 선점형이 아니면 while문을 회전하며 기다리는 쓰레드가 영원히 CPU를 독점하게 된다.

### 스핀 락 평가

위 방식은 우선 상호 배제를 만족한다. 그러나 공정하지는 않다. 스핀 락은 어떤 공정성도 보장해주지 않는다. 쓰레드가 경쟁에 밀려서 계속 락을 대기하는 상황이 발생할 수도 있다. 그럼 성능은 어떨까? 단일 프로세서와 멀티 프로세서의 환경에서 각각 생각해보자.

단일 CPU의 경우에 락을 획득한 쓰레드를 제외한 N-1개의 쓰레드가 있다고 할 때, N-1개의 쓰레드들이 차례대로 스케줄러에 의해 실행되는 동안 CPU 사이클을 낭비하게 된다. 반면에 멀티 CPU의 경우에는 꽤 합리적으로 동작한다. 쓰레드 A는 CPU 1에, 쓰레드 B는 CPU 2에서 락을 획득하기 위해 경쟁 중이라고 하자. A가 먼저 락을 획득하고 B가 대기하는 상황에서 B는 CPU 2에서 락을 대기한다. 임계 영역의 구간이 매우 짧다고 하면 B가 금방 락을 획득할 수 있다. 

잘 생각해보면, 단일 프로세서에선 락을 대기하는 쓰레드가 실행될 때, 기존 락 소유자가 락을 해제하지 못하니까 문제가 생기지만 멀티 프로세서에선 락을 대기하는 쓰레드가 다른 CPU에서 실행되기 때문에 기존 락 소유자가 금방 락을 해제할 수 있어 문제가 안 생기는 것이다. 

### Compare-And-Swap

```c
int CompareAndSwap(int *ptr, int expected, int new){
	int actual = *ptr;
	if(actual == expected)
		*ptr = new;
	return actual;
}
```

또 다른 하드웨어 기법인 Compare-And-Swap을 보자. 변수의 값이 기댓값과 일치하면 새로운 값으로 변경한다. 원래의 값을 반환하여 락 획득 성공 여부를 알 수 있다. 이전 Test-And-Set과 같은 방식으로 락을 구현할 수 있다.

```c
void lock(lock_t *lock){
	while(CompareAndSwap(&lock->flag, 0, 1) == 1) ;
}
```

락을 대기하는 쓰레드는 while 문에서 회전하게 된다. CompareAndSwap은 TestAndSet보다 더 강력하다. 대기없는 동기화를 다룰 때 더 진가를 발휘한다.

(추가로, Load-Linked, Store-Conditional을 같이 쓰는 방법도 있는데 자세한 내용은 생략..)

### Fetch-And-Add

```c
int FetchAndAdd(int *ptr){
	int old = *ptr;
	*ptr = old + 1;
	return old;
}
```

기존 값을 반환하면서 증가시키는 원자적 연산이다. 이 명령어를 통해 **티켓 락** 구현이 가능하다.

```c
typedef struct _ _lock_t {
	int ticket;
	int turn;
} lock_t;

void lock_init(lock_t *lock) {
  lock−>ticket = 0;
	lock−>turn = 0;
}

void lock(lock_t *lock) {
	int myturn = FetchAndAdd(&lock−>ticket);
  while (lock−>turn != myturn) ; 
}

void unlock(lock_t *lock) {
	FetchAndAdd(&lock−>turn);
}
```

lock→turn을 통해 어느 쓰레드의 차례인지 판단한다. 만약 내 차례가 오면 myturn == turn 조건이 성립할 때, 임계 영역이 진입할 수 있다. 이는 이전과는 다르게 모든 쓰레드들이 차례로 임계 영역에 접근할 수 있어 공정함을 보장할 수 있다는 장점이 있다.

### 과도한 스핀

앞서 말한 하드웨어 기반의 락은 상호 배제를 보장하며 제대로 동작한다. 그러나 성능 상의 문제가 있는데 단일 프로세서 환경에서 락을 대기하는 쓰레드가 락을 획득하기 위해 기다릴 때 CPU가 계속 낭비 된다. 이 문제를 어떻게 해결할 수 있을까? **이는 운영체제의 도움이 필요하다.**

### 무조건 양보!

가장 간단한 방법으로 락이 해제되기를 기다리며 스핀해야 하는 경우에는 무조건 CPU를 다른 쓰레드에게 양보한다. 

```c
void init() { flag = 0; }

void lock() {
	while(TestAndSet(&flag, 1) == 1)
		yield();
}

void unlock() { flag = 0; }
```

**yield() 시스템 콜은 호출 쓰레드의 상태를 실행에서 준비로 변환하여 다른 쓰레드가 실행되게 한다.** 단일 프로세서라고 할 때, CPU를 최대한 빠르게 락 소유자에게 양보해 락을 해제하게 만드는 것이다.

그런데 만약 100개의 쓰레드가 실행된다고 하자. 그럼 락을 획득한 1개의 쓰레드를 제외하고 나머지 99개의 쓰레드가 실행하고 양보하는 패턴이 반복되며 문맥 교환, CPU 낭비 등의 비용이 여전히 발생한다.

### 잠자기

그래서 우리는 **다음에 어떤 쓰레드가 락을 획득할 지를 명시적으로 제어할 수 있어야 한다.** 무조건 양보 방식은 다음 쓰레드를 선택할 수 없어 최악의 경우에 비용이 너무 많이 낭비된다. 이를 구현하기 위해 호출한 쓰레드를 재우는 park()와 특정 쓰레드를 깨우는 unpark(threadID)를 사용한다. 이와 함께 TestAndSet을 함께 사용하여 더욱 효율적인 락을 만들어보자.

```c
typedef struct __lock_t {
	int flag;
	int guard;
	queue_t *q;
} lock_t;

void lock_init(lock_t *m){
	m->flag=0;
	m->guard=0;
	queue_init(m->q);
}

void lock(lock_t *m){
	while(TestAndSet(&m->guard, 1)==1) ;
	if(m->flag==0){
		m->flag=1;
		m->guard=0;
	} else {
		queue_add(m->q, gettid());
		m->guard=0;
		park();
	}
}

void unlock(lock_t *m){
	while(TestAndSet(&m->guard, 1)==1);
	if(queue_empty(m->q))
		m->flag=0;
	else
		unpark(queue_remove(m->q));
	m->guard=0;
}
```

여기서 만약 이미 락을 획득한 쓰레드가 있다고 할 때, 다른 쓰레드가 park()를 호출하기 전에 락 소유자에게 CPU가 할당 되어 이 쓰레드가 락을 해제한 후 다시 원래 쓰레드로 돌아와 park()를 호출하면 어떻게 될까? 이 쓰레드는 깨어날 방법이 없다. 이 문제는 **깨우기/대기 경쟁(wakeup/waiting race)** 라고도 불린다.

이 문제는 setpark()를 통해 해결하는데 이 루틴은 park()를 호출하기 전에 다른 쓰레드가 unpark()를 먼저 호출한다면 추후 park()는 잠을 자는 대신 바로 리턴된다.

```c
queue_add(m->q, gettid());
setpark();
m->guard=0;
```

바뀐 부분은 위와 같다.


