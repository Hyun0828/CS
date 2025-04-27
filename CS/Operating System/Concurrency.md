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
- [Condition Variables](#30-condition-variables)
  - [컨디션 변수](#컨디션-변수)
  - [생산자/소비자 (유한 버퍼) 문제](#생산자소비자-유한-버퍼-문제)
  - [컨디션 변수 주의점](#컨디션-변수-주의점) 

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

## 30. Condition Variables

‘락’만으로는 제대로된 병행 프로그램을 작성할 수 없다. 쓰레드가 계속 진행하기 전에 어떤 조건이 참인지를 검사해야 하는 경우가 많이 있다. 대표적으로 join()이 있다. 공유 변수를 두어 join을 구현하게 되면 while문을 통해 스핀 락처럼 계속 회전하며 CPU 사이클을 낭비하게 되어 다른 방법이 필요하다.

```c
volatile int done = 0;

void *child(void *arg){
	printf("child\n");
	done = 1;
	return NULL;
}

int main(int argc, char *argv[]){
	printf("parent : begin\n");
	pthread_t c;
	Pthread_create(&c, NULL, child, NULL);
	while(done==0) ;
	printf("parent: end\n");
	return 0;
}
```

### 컨디션 변수

컨디션 변수는 일종의 큐로서 어떤 조건이 만족될 때까지 기다리며 쓰레드가 대기할 수 있는 큐이다. 다른 쓰레드가 상태를 변경 시켰을 때 대기 중이던 쓰레드를 깨우고 계속 진행할 수 있게 만든다. 컨디션 변수에는 wait()과 signal() 연산이 있는데 wait()는 호출 쓰레드를 잠재우고 signal()은 대기 중인 쓰레드를 깨운다. 이 때 주의할 점은 wait() 연산 시 mutex를 매개변수로 사용한다는 것이다.

wait()으로 쓰레드가 잠들 때 들고 있는 락을 해제한다. 또한 **대기 상태의 쓰레드가 깨어나면 wait()를 리턴하기 전에 락을 재획득해야 한다.** 이렇게 구성된 이유는 경쟁 조건의 발생을 방지하기 위함이다.

 

```c
int done = 0;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t c = PTHREAD_COND_INITIALIZER;

void thr_exit() {
	Pthread_mutex_lock(&m);
	done = 1;
	Pthread_cond_signal(&c);
	Pthread_mutex_unlock(&m);
}

void *child(void *arg) {
	printf(“child\n ”);
	thr_exit();
	return NULL;
}

void thr_join() {
	Pthread_mutex_lock(&m);
	while (done == 0)
		Pthread_cond_wait(&c, &m);
	Pthread_mutex_unlock(&m);
}

int main(int argc, char *argv[]) {
	printf(“parent: begin\n ”);
	pthread_t p;
	Pthread_create(&p, NULL, child, NULL);
	thr_join();
	printf(“parent: end\n ”);
	return 0;
}
```

위의 코드에서 2가지 경우가 발생한다.

1. 부모 쓰레드에서 join을 먼저 실행하는 경우에는 부모 쓰레드가 먼저 잠들고 자식 쓰레드가 done을 1로 바꾸면서 부모 쓰레드를 깨운다. 
2. 자식 쓰레드가 먼저 done을 1로 바꾸고 깨우는데 대기하는 쓰레드가 없다. 이후 부모 쓰레드가 Join을 실행하면 바로 리턴한다.

여기서 중요한 사실은 **join에서 조건을 검사할 때 꼭 if문이 아닌 while문을 사용해야 한다는 것이다.** 이유는 나중에 다시 생각해보자.

위의 코드에서 done 이라는 상태 변수나 mutex_lock 둘 중 어느 하나라도 사용하지 않으면 코드가 정상 동작하지 않는다.

### 생산자/소비자 (유한 버퍼) 문제

흔히 **Producer/Consumer** 문제라고도 부른다. 생산자는 데이터를 만들어 버퍼에 넣고 소비자는 버퍼에서 데이터를 꺼내 사용한다. 웹 서버의 경우에도 HTTP 요청을 작업 큐에 넣고 소비자 쓰레드가 큐에서 요청을 꺼내 처리한다. 이 유한 버퍼는 공유 자원이기 때문에 경쟁 조건 발생을 막기 위해서는 동기화가 필요하다. 버퍼를 간단한 값이라고 생각하고 예제를 보자.

```c
int buffer;
int count = 0;

void put(int value){
	assert(count==0);
	count=1;
	buffer=value;
}

int get(){
	assert(count==1);
	count=0;
	return buffer;
}
```

간단히 put은 버퍼가 비어있으면 값을 넣고 count를 1로 바꾼다. get은 버퍼에 값이 있으면 count를 0으로 만들고 버퍼 값을 리턴한다. 이 말은 즉 버퍼가 꽉 차있을 때 데이터를 넣어선 안 되며 비어있을 때 값을 꺼내서도 안 된다.

우선 생산자와 소비자 쓰레드가 1개씩 존재한다고 가정해보자. 버퍼 접근 자체가 임계 영역이므로 락의 추가와 상태 변수를 같이 사용해야 한다.

```c
cond_t cond;
mutex_t mutex;

void *producer(void *arg) {
	int i;
	for (i = 0; i < loops; i++) {
		Pthread_mutex_lock(&mutex); // p1
		if (count == 1) // p2
			Pthread_cond_wait(&cond, &mutex); // p3
		put(i); // p4
		Pthread_cond_signal(&cond); // p5
		Pthread_mutex_unlock(&mutex); // p6
	}
}

void *consumer(void *arg) {
	int i;
	for (i = 0; i < loops; i++) {
		Pthread_mutex_lock(&mutex); // c1
		if (count == 0) // c2
			Pthread_cond_wait(&cond, &mutex); // c3
		int tmp = get(); // c4
		Pthread_cond_signal(&cond); // c5
		Pthread_mutex_unlock(&mutex); // c6
		printf(“%d\n ”, tmp);
	}
}
```

생산자와 소비자가 1개면 위의 코드는 정상 동작한다. **그러나 쓰레드가 여러개면 무슨 문제가 생길까?**

**첫 번째 문제점은 if 문과 관련이 있다.** 예를 들어 생산자가 1, 소비자가 2개라고 하자. 소비자1가 먼저 실행되면 대기상태로 전환되고 생산자1이 실행되면 소비자1을 깨우면서 버퍼에 데이터를 넣는다. 이 때 소비자1은 준비 상태로 전환되는데 소비자2가 먼저 실행되면 소비자2가 버퍼에서 데이터를 꺼내 사용한다. 이후 소비자1이 실행되면 wait 이후 코드를 실행하게 되는데 버퍼는 비워져 있기 때문에 get()을 할 수가 없다.

즉, 깨어난 쓰레드가 실제 실행되는 시점에도 그 상태가 유지된다는 보장이 없는데 이런 식의 시그널 정의를 **Mesa semantic** 이라고 한다. 반대로 깨어난 즉시 쓰레드가 실행되는 개념이 Hoare semantic 이지만 구현이 어려워 전자가 사용되고 있다.

즉 앞서 말한 것처럼 조건문에는 If문이 아니라 while문을 사용하면 문제를 해결할 수 있다. while문이면 소비자1이 깨어나도 조건을 다시 확인하기 때문에 잠들어 문제가 발생하지 않는다. **컨디션 변수에는 반드시 while문을 사용하도록 하자.** 그러나 아직도 문제가 있다. 

```c
cond_t cond;
mutex_t mutex;

void *producer(void *arg) {
	int i;
	for (i = 0; i < loops; i++) {
		Pthread_mutex_lock(&mutex); // p1
		while (count == 1) // p2
			Pthread_cond_wait(&cond, &mutex); // p3
		put(i); // p4
		Pthread_cond_signal(&cond); // p5
		Pthread_mutex_unlock(&mutex); // p6
	}
}

void *consumer(void *arg) {
	int i;
	for (i = 0; i < loops; i++) {
		Pthread_mutex_lock(&mutex); // c1
		while (count == 0) // c2
			Pthread_cond_wait(&cond, &mutex); // c3
		int tmp = get(); // c4
		Pthread_cond_signal(&cond); // c5
		Pthread_mutex_unlock(&mutex); // c6
		printf(“%d\n ”, tmp);
	}
}
```

**두 번째 문제는 컨디션 변수의 개수가 1개라는 점이다.** 이전과 달리 소비자 쓰레드 2개가 먼저 실행된다고 하자. 소비자 쓰레드 2개는 잠들고 생산자 쓰레드가 실행되면서 소비자 쓰레드 중 1개를 깨운다. 소비자1이 깨어나면서 버퍼에서 값을 꺼낸다. 이후 시그널을 통해 소비자2나 생산자1 중 하나를 깨우는데 만약에 소비자2를 깨우면 모든 쓰레드가 잠들게 되어 깨워줄 쓰레드가 없어진다. 그래서 우리는 컨디션 변수를 1개 더 만들어 생산자는 소비자를, 소비자는 생산자만을 깨울 수 있게 만든다.

```c
cond_t empty, fill;
mutex_t mutex;

void *producer(void *arg) {
	int i;
	for (i = 0; i < loops; i++) {
		Pthread_mutex_lock(&mutex); // p1
		while (count == 1) // p2
			Pthread_cond_wait(&empty, &mutex); // p3
		put(i); // p4
		Pthread_cond_signal(&fill); // p5
		Pthread_mutex_unlock(&mutex); // p6
	}
}

void *consumer(void *arg) {
	int i;
	for (i = 0; i < loops; i++) {
		Pthread_mutex_lock(&mutex); // c1
		while (count == 0) // c2
			Pthread_cond_wait(&fill, &mutex); // c3
		int tmp = get(); // c4
		Pthread_cond_signal(&empty); // c5
		Pthread_mutex_unlock(&mutex); // c6
		printf(“%d\n ”, tmp);
	}
}
```

그러나 우리가 변경해야 할 점이 하나 더 있다. 병행성을 증가시키는 것인데 버퍼 공간을 추가하여 대기 상태에 들어가기 전에 여러 값들이 생산될 수 있게, 소비될 수 있게 하는 것이다. 

```c
int buffer[MAX];
int fill = 0;
int use = 0;
int count = 0;

void put(int value){
	buffer[fill] = value;
	fill = (fill + 1) % MAX;
	count++;
}

int get(){
	int tmp = buffer[use];
	use = (use + 1) % MAX;
	count--;
	return tmp;
}
```

그냥 버퍼 크기를 늘리고, 버퍼가 꽉 차면 생산자가 잠들고 버퍼가 비어있으면 소비자가 잠든다.

### 컨디션 변수 주의점

이전에 이야기 했던 것처럼 조건문을 사용할 때는 반드시 if문이 아닌 while문을 사용해야 한다. 쓰레드를 잘못 깨운 경우를 대비하기 위함이다. 또한 멀티 쓰레드 기반 메모리 할당 예제에서도 이 이슈가 등장한다. 메모리 할당 시 여유 공간이 생길 때까지 대기하고 쓰레드가 메모리 반납 시 시그널을 생성해 쓰레드를 깨우는데 이 때 어떤 쓰레드를 깨워야 할까?

```c
int bytesLeft = MAX_HEAP_SIZE;
cond_t c;
mutex_t m;

void *allocate(int size) {
	Pthread_mutex_lock(&m);
	while (bytesLeft < size)
		Pthread_cond_wait(&c, &m);
	void *ptr = . . . ; 
	bytesLeft −= size;
	Pthread_mutex_unlock(&m);
	return ptr;
}

void free(void *ptr, int size) {
	Pthread_mutex_lock(&m);
	bytesLeft += size;
	Pthread_cond_signal(&c); 
	Pthread_mutex_unlock(&m);
}
```

쓰레드a가 100바이트 공간을 대기, 쓰레드b가 10바이트 공간을 대기할 때 쓰레드c가 50바이트 공간을 반납하면 우리는 쓰레드b를 깨워야 한다. 그러나 어떤 쓰레드를 깨워야할 지 모르기 때문에 **pthread_cond_broadcast()를 통해 모든 쓰레드를 다 깨우는 방법을 선택한다.** 물론 성능에 안 좋은 영향을 끼칠 수도 있다. 더 좋은 해법이 있다면 그 방법을 선택하면 된다. 이전에는 컨디션 변수를 하나 더 두어 문제를 해결한 케이스다.
