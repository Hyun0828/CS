Spring에서 비동기 처리를 위해서 @Async 어노테이션을 사용하는데 기본적으로는 SimpleAsyncTaskExecutor가 사용된다. 이는 비동기 작업마다 새로운 스레드를 생성하는 스레드 풀이기 때문에 리소스 낭비, 성능 저하, 스케일링 문제 등이 발생할 수 있다.

이와 같은 이유로 보통은 Java에서 제공하는 ThreadPoolExecutor를 Wrapping한, TaskExecutor interface의 구현체인 **ThreadPoolTaskExecutor**를 새롭게 Bean 객체로 정의해서 사용한다.

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "threadPoolTaskExecutor")
    public Executor getTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("Thread-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

여기서 각 설정 값에 대해 잘못 알고 있었던 부분들이 있어 정리해보려고 한다.

1. **사전에 스레드 풀의 크기만큼 (setCorePoolSize) 스레드가 자동으로 생성되지 않는다.**
    - 위와 같은 설정에서 애플리케이션 실행 시 스레드 풀에 2개의 스레드가 자동으로 생성될 것으로 생각할 수 있으나 그렇지 않다. 실제로는 스레드 작업이 요청될 때마다 필요한 스레드가 생성되기 시작한다.
    - setPrestartallCoreThreads 메소드 설정을 통해 시작부터 일정 스레드 개수를 생성할 수 있다

1. **CorePoolSize를 초과하더라도 곧바로 MaxPoolSize만큼 스레드 개수를 늘리지 않는다.**
    - CorePoolSize만큼 스레드가 생성되고 이후 작업 요청에 대한 스레드는 Queue에 저장되며 queueCapacity 만큼, Queue가 가득 차면 maxPoolSize 만큼의 스레드 개수 증가가 이루어진다.
    - queueCapacity는 기본적으로 Integer.MAX_VALUE 값으로 설정되어 있기 때문에 커스텀 설정을 해주지 않으면 사실상 스레드 풀의 크기가 corePoolSize에서 머물게 된다.

1. **ExecutionHandler에 대해 살펴보자**
   
   스레드 개수가 maxPoolSize + queueCapacity 값을 초과하게 되면 **rejectedExecutionException**이 발생하게 되는데 이를 처리하는 정책은 4가지가 있다.
    1. **AbortPolicy** : 예외를 던지며 기본 정책이다.
    2. **CallerRunsPolicy** : 현재 호출한 스레드가 직접 실행한다. 즉 Async 메소드를 호출한 스레드 (보통은 메인 스레드)가 처리하게 되며 실질적으로 동기 처리로 이루어진다. 그러나 task 유실은 발생하지 않아 안정적이다.
    3. **DiscardPolicy** : 작업을 버린다.
    4. **DiscardOldestPolicy** : 가장 오래된 작업을 버리고 새 작업을 추가한다.
