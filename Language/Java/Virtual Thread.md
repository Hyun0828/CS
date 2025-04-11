## Virtual Thread란?

![image](https://github.com/user-attachments/assets/7be171e7-e8a4-4e95-a528-f7652e5ce52c)


Java의 전통적인 Thread는 JVM 내에서 생성된 Platform Thread(User Thread)가 Java Native Interface(JNI)를 통해 OS의 Kernel Thread와 1대1 매핑되도록 설계되었다.

![image](https://github.com/user-attachments/assets/cc7dcb24-d511-42a2-b532-b3addcbb8d5e)


Java의 스레드는 I/O, Interrupt, sleep과 같은 상황에서 block/waiting 상태가 되는데 이 때 다른 스레드가 커널 스레드를 점유하여 작업을 수행하는 것을 ‘**컨텍스트 스위치**’라고 부른다.

즉 플랫폼 스레드가 대기 상태로 접어들면 그에 대응되는 커널 스레드가 대기 상태로 바뀌고 다른 커널 스레드가 실행 상태로 바뀌는 것을 의미한다.

이런 전통적인 스레드는 프로세스 내의 공통 부분은 공유하여 공통영역을 제외하고 생성되기 때문에 프로세스에 비해 크기가 작아 생성 비용이 적고 프로세스에 비해 컨텍스트 스위치 비용이 저렴해 많이 사용되었다.

그러나 요청량이 갈수록 많아지는 서버에서는 더 많은 스레드 수를 요구하게 되었고 보통 스레드 1개가 메모리 1MB를 차지하는데 4GB RAM을 사용하면 많아야 4,000개의 스레드를 생성할 수 밖에 없다. 또한 스레드가 많아지면서 컨텍스트 스위칭 비용도 기하급수적으로 늘어나는 문제가 발생했다.

이 문제를 해결하기 위해 경량 스레드 모델인 **Virtual Thread**가 등장했다.

## Virtual Thread의 구조

![image](https://github.com/user-attachments/assets/2f5d77fb-34e6-440b-a230-28baa2ea828d)


Virtual Thread는 기존 스레드와 달리 플랫폼 스레드 위에서 여러 Virtual Thread가 번갈아 가며 실행되는 형태로 동작한다. 좋은 점은 **Virtual Thread는 컨텍스트 스위칭 비용이 저렴하다는 것**이다.

|  | **Thread** | **Virtual Thread** |
| --- | --- | --- |
| Stack 사이즈 | ~2MB | ~10KB |
| 생성시간 | ~1ms | ~1µs |
| 컨텍스트 스위칭 | ~100µs | ~10µs |

또한 Thread는 기본적으로 최대 2MB의 스택 메모리 사이즈를 가지기 때문에 컨텍스트 스위칭 시 메모리 이동량이 크다. 또한 스레드 생성을 위해서는 커널 스레드와의 매핑을 위해 시스템 콜을 호출하는데 이 비용도 만만치 않다.

하지만 Virtual Thread는 JVM에 의해 생성되기 때문에 시스템 콜과 같은 커널 영역의 호출이 적고 메모리 크기가 일반 스레드에 비해 매우 작기 때문에 컨텍스트 스위칭 비용 또한 매우 작다는 장점이 있다.

우선 Platform Thread의 기본 스케줄러는 **ForkJoinPool**이다. 스케줄러는 Platform Thread Pool을 관리하고 Virtual Thread의 작업 분배 역할을 한다.

![image](https://github.com/user-attachments/assets/9969c29b-7f91-4992-b73e-dd3a664b809a)


Virtual Thread를 살펴보면

- carrierThread를 가지고 있다. 실제로 작업을 수행시키는 platform thread를 의미하고 workQueue를 가지고 있다.
- scheduler라는 ForkJoinPool을 가지고 있다. carrier thread의 pool 역할을 하며 virtual thread의 작업 스케줄링을 담당한다.
- runContinuation이라는 virtual thread의 실제 작업 내용(Runnable)을 가지고 있다.

## Virtual Thread의 동작 원리

![image](https://github.com/user-attachments/assets/5f418882-5444-4ad8-9bac-f31375c21018)


- 실행될 Virtual Thread의 작업인 runContinuation을 carrier thread의 workQueue에 push한다.
- workQueue에 있는 runContinuation들은 forkJoinPool에 의해 work stealing 방식으로 carrier thread에 의해 처리된다.
- 처리되던 runContinuation들을 I/O, sleep으로 인한 interrupt나 작업 완료 시, work queue에서 pop되어 park과정에 의해 다시 힙 메모리로 돌아간다.

→ 이 때 virtual thread가 꼭 특정 carrier thread와 고정 매핑되어 있는 것이 아니라, 처음 생성될 때는 특정 carrier thread에 바인딩 되지만 실행이 끝나거나 I/O를 만난 이후 다시 실행 될 때는 다른 carrier thread에 바인딩 될 수도 있다. 따라서 어떤 virtual thread의 runContinuation을 특정 carrier thread에 스케줄링 하는 것이 forkJoinPool 같은 executor의 역할이다. 기본적으로 virtual thread는 kernel thread와 바인딩 되지 않고 OS 리소스를 거의 안 쓰며 작업 스케줄링이 JVM 안에서 처리되기 때문에 성능적으로 우수하다.

## Virtual Thread 성능 비교

직접적으로 성능 테스트를 해본 건 아니지만, 여러 빅테크 기업의 기술 블로그를 참조했을 때

- 일반 Thread 모델에 비해 I/O Bound 작업에서는 큰 성능 향상이 있었으나 CPU Bound 작업에는 결국 경량 Thread도 일반 Thread 위에서 동작하기 때문에 경량 Thread가 Switching 되지 않는 경우에는 경량 Thread 생성 및 스케줄링 비용 등의 낭비가 발생하여 오히려 성능이 나빠진다고 한다.
- WebFlux 같은 Reactive Programming 모델과 비교했을 때도 I/O Bound 작업에서 큰 성능 향상을 보였다고 한다.

## 주의사항

- **No pooling**
    
    Virtual Thread는 생성 비용이 작기 때문에 스레드 풀을 만들기 보단 필요할 때마다 생성하고 GC에 의해 소멸되도록 방치하는 것이 좋다.
    
- **CPU Bound 작업엔 비효율**
    
    CPU 작업을 수행할 땐 오히려 일반 스레드를 사용하는 것이 좋다.
    
- **Pinned Issue**
    
    Virtual Thread 내에서 synchronized나 JNI를 통해 네이티브 메서드를 쓰면 Virtual Thread가 Carrier Thread에 park 될 수 없는 상태(Virtual Thread가 Carrier Thread에 고정되어 버리는 상태)가 되어버리는데 이를 Pinned 상태라고 한다. 이는 큰 성능 저하를 유발할 수 있다.
    
    아직 Spring 진영에서는 ssynchronized를 사용하는 라이브러리가 많아서 ReentrantLock으로 마이그레이션 하고 있다고 한다.
    
- **Thread Local (Thread 전용 데이터를 저장할 수 있는 저장소)**
    
    Virtual Thread는 수백만개의 스레드를 운용할 수 있도록 설계되었기 때문에 항상 크기를 작게 유지하는 것이 좋다.
    

---

### 참고 블로그

https://tech.kakaopay.com/post/ro-spring-virtual-thread/

https://techblog.woowahan.com/15398/

https://d2.naver.com/helloworld/1203723
