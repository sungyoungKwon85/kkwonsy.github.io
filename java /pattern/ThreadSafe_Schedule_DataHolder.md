# ThreadSafe & 스케쥴링 & repository 데이터 holder
## 요약
- interface Task<T> 
- interface CallbackTask<T> extends Task<T>
- interface SchedulingCallbackTask<T> extends CallBackTesk<T>
- abstract class AbstractSchedulingTaskExecutor<T> implements SchedulingCallbackTask<T>
> @ThreadSafe
- interface HoldingResultTask<T> extends Task<T>
- abstract class AbstractHoldingResultSchedulingTaskExecutor<T> extends AbstractSchedulingTaskExecutor<T> implements HoldingResultTask<T>
> @ThreadSafe

## 순서
1. 로딩시 레포지토리에서 데이터를 들고올 task를 실행해 holder에 담아두기
2. SchedulingCallbackTask 구현체를 스케쥴러로 등록하기 
3. 요청 들어올 때 holder에서 꺼내 사용하기 


## 코드  
- Task
```
public interface Task<T> {
    @Nullable
    T task();
}
```

<br>

- CallbackTask
```
public interface CallbackTask<T> extends Task<T> {
    void callback(@Nonnull T result);
}
```

<br>

- SchedulingCallbackTask  
```
public interface SchedulingCallbackTask<T> extends CallbackTask<T> {
    Trigger schedule();
    void execute();

    default Trigger createEveryFiveMinutesTrigger() {
        return new CronTrigger("0 */5 * * * *");
    }

    default Trigger createEveryTenMinutesTrigger() { // (11) 10분마다 도는 스케쥴링 셋팅 
        return new CronTrigger("0 */10 * * * *");
    }
}
```
<br>

- AbstractSchedulingTaskExecutor
```
@ThreadSafe
@Slf4j
public abstract class AbstractSchedulingTaskExecutor<T> implements SchedulingCallbackTask<T> {

    private final AtomicReference<Future<Void>> taskHolder = new AtomicReference<>();

    @Override
    public void execute() {
        Future<Void> runnableTask = taskHolder.get();
        if (runnableTask == null) {
            FutureTask<Void> taskToBeExecute = new FutureTask<>(() -> {
                try {
                    T result = task(); // (3) MyAService.task 실행 
                    if (result != null) {
                        callback(result); // (5) 
                    }
                } catch (Exception e) {
                    log.error("스케쥴링 중 오류 발생", e);
                } finally {
                    taskHolder.set(null);
                }
                return null;
            });

            synchronized (this) {
                if (taskHolder.get() == null) {
                    taskHolder.set(taskToBeExecute);
                } else {
                    runnableTask = taskHolder.get();
                }
            }

            if (runnableTask == null) {
                runnableTask = taskToBeExecute;
                taskToBeExecute.run();
            }
        }

        try {
            runnableTask.get();
        } catch (InterruptedException | ExecutionException e) {
            log.warn("스케쥴링 수행 중 예외 발생", e);
        }
    }
}
```
<br>

- HoldingResultTask
```
public interface HoldingResultTask<T> extends Task<T> {

    T getResult();

    default T getResultOrDefault(@Nonnull T defaultValue) {
        T value;
        return  (value = getResult()) == null ? defaultValue : value;
    }
}
```
<br>

- AbstractHoldingResultSchedulingTaskExecutor
```
@ThreadSafe
public abstract class AbstractHoldingResultSchedulingTaskExecutor<T> extends AbstractSchedulingTaskExecutor<T> implements HoldingResultTask<T> {

    private final AtomicReference<T> resultHolder = new AtomicReference<>();

    @Override
    public final void callback(@Nonnull T result) { // (6) 레포지토리의 데이터를 resultHolder에 넣는다. 
        resultHolder.set(result);
    }

    @Override
    public T getResult() { // (15) 레포지토리 데이터가 담긴 resultHolder를 들고온다 
        return resultHolder.get();
    }

    public T getResultOrDefaultThenExecuteIfEmpty(@Nonnull T defaultValue) {
        T value;
        return (value = getResultOrExecuteIfEmpty()) == null ? defaultValue : value;
    }

    @Nullable
    public T getResultOrExecuteIfEmpty() {

        if (getResult() == null) {
            execute();
        }

        return getResult();
    }
}
```
<br>

- SchedulingConfiguration
```
@Configuration
@RequiredArgsConstructor
public class SchedulingConfiguration implements SchedulingConfigurer {

    private final ApplicationContext applicationContext;

    @Bean
    @Qualifier("threadPoolTaskScheduler")
    public ThreadPoolTaskScheduler threadPoolTaskScheduler() {
        ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
        taskScheduler.setPoolSize(5);
        taskScheduler.setThreadNamePrefix("taskScheduler-");
        return taskScheduler;
    }

    @Override
    public void configureTasks(@Nonnull ScheduledTaskRegistrar scheduledTaskRegistrar) {
        scheduledTaskRegistrar.setTaskScheduler(threadPoolTaskScheduler());
    }

    @EventListener(ApplicationReadyEvent.class) // (7) 애플리케이션이 기동하면서 실행된다. 
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public void executeAllSchedule() throws BeansException {
        TaskScheduler taskScheduler = threadPoolTaskScheduler();

        for (SchedulingTask<?> schedulingTask : applicationContext.getBeansOfType(SchedulingTask.class).values()) {
            Trigger trigger = schedulingTask.getSchedule();
            taskScheduler.schedule(schedulingTask::executeSchedule, trigger);
            if (!(schedulingTask instanceof BasedModifiedDateTimeRefreshEventTask) && trigger instanceof CronTrigger) {
                schedulingTask.executeSchedule();
            }
        }

        // (8) SchedulingCallbackTask 를 구현한 Task를 스케쥴로 등록한다. 
        for (SchedulingCallbackTask schedulingTask : applicationContext.getBeansOfType(SchedulingCallbackTask.class).values()) {
            taskScheduler.schedule(schedulingTask::execute, schedulingTask.schedule());  // (9) MyAService.schedule() 호출 
        }
    }
}
```
<br>

- MyAService
```
@Service
@Slf4j
public class MyAService extends AbstractHoldingResultSchedulingTaskExecutor<List<MyAEntity>> implements MyAService {
    private final myARepository myARepository;
    private final TransactionTemplate readOnlyTransactionTemplate;

    @EventListener(ApplicationReadyEvent.class) // (1) 애플리케이션 기동하면서 실행된다. 
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public void afterApplicationIsReady() {
        execute(); // (2) AbstractHoldingResultSchedulingTaskExecutor -> AbstractSchedulingTaskExecutor.execute() 실행 
    }

    @Override
    public final List<MyAEntity> task() { // (4) 레포지토리에서 데이터를 들고온다. 
        return readOnlyTransactionTemplate.execute(status -> {
            List<myAEntity> result = myARepository.findAll();
            log.info("===> tb_my_a preloaded {}", result);
            return result;
        });
    }

    @Override
    public final Trigger schedule() {
        return createEveryTenMinutesTrigger(); // (10) 10분마다 도는 테스크로 만든다. 
    }

    @Override
    public List<MyAEntity> getMyAs() { // (14) 
        return getResult();
    }
```
<br>

- MyAService
```
public interface MyAService {

    List<MyAEntity> getMyAs();

    default Map<Long, MyAEntity> getAIdAndMyAMap() { // (13) 
        return getMyAs().stream()
                .collect(Collectors.toMap(MyAEntity::getAId, Function.identity(), (n,o) -> n));
    }
}
```
<br>

- MyBServiceImpl
```
@Service
@Slf4j
@RequiredArgsConstructor
public class MyBServiceImpl {
  private final MyAService myAService;

  private ABCDto getMyA(Predicate<ABCDto> predicate){
      Map<Long, MyAEntity> map = myAService.getAIdAndMyAMap(); // (12) 어떤 요청이 들어와서 실행
      return ...;
  }
}
```