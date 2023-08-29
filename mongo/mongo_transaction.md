# Transaction 

- 트랜잭션은 데이터베이스의 논리적 처리 그룹임.  
> 몽고는 4.2부터 ACID 를 지원한다.  

<br>

- 몽고는 트랜잭션을 사용하기 위한 두가지 API를 제공한다  
> core api, callback api  
> core api: 트랜잭션 시작, 오류에 재시도 등을 개발자가 직접 작성해야 함.  
> callbak api:많은 기능을 wrapping한 단일 함수를 제공한다.  


<br>


- 몽고 트랜잭션에는 두 가지 주요 제한 범주가 있다.  
> 트랜잭션의 시간 제한  
>> transactionLifetimeLimitSeconds 로 수정 가능  
>> 샤드가 있으면 모든 노드에 설정해야 함  
>> 최대 60초와 transactionLifetimeLimitSeconds / 2 중 낮은 값을 택함  

>> 트랜잭션 작업에 필요한 락을 획득하기 위해 대기하는 시간은 최대 5ms이다.  
>> maxTransactionLockRequestTimeoutMillis로 수정가능  

<br>

> 몽고 oplog 항목과 개별 항목에 대한 크기 제한  
>> 몽고는 트랜잭션의 쓰기 작업에 필용한 만큼 oplog 항목을 생성한다.  
>> BSON 크기 제한인 16MB까지만.  

<br>

---

<br>

몽고는 유연성 있는 모델과 스키마 설계 패턴과 같은 모범 사례를 사용하면 굳이 트랜잭션을 쓰지 않아도 된다.  



