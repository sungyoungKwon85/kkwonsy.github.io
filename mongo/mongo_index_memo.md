## 인덱싱 
> 인덱스를 사용하지 않는 쿼리를 컬렉션 스캔이라 한다.  
> 성능을 위해 인덱스를 사용해야 한다   
> 단점? CUD작업은 인덱스를 수정해야 하기 때문에 READ보다 CUD가 훨씬 많다면 다시 생각해봐야함.   
```
  .explain("executionStats")을 사용하면 성능을 확인해볼 수 있다. 
    살펴보면 좋은 필드. 
    nReturned
    totalDocsExamined
    executionTimeMillis 
```
> 어떤필드?  
> 자주쓰는 쿼리를 조사해 공통적인 키 셋을 찾아볼 것.   

### 복합인덱스?
> 상당 수 쿼리 패턴은 두 개 이상의 키를 기반으로 인덱스를 작성한다   
> db.users.find().sort({"age":1, "username": 1}) 이 많이 사용된다면   
> age와 username을 이용해 인덱스를 만든다.   
> **순서가 중요하다.**   
```
db.users.createIndex({"age":1, "username": 1})
```

> 스캔할 레코드 개수를 인덱스가 얼마나 최소화 하느냐가 중요함.   
> 동등 필터에 대한 키를 맨 앞에 표시해야 한다.   
> 정렬에 사용되는 키는 다중값 필드 앞에 표시해야 한다   
> 다중값 필터에 대한 키는 마지막에 표시해야 한다   
> *[복합인덱스설계예제](https://github.com/sungyoungKwon85/sungyoungKwon85.github.io/blob/main/mongo/multi_index.md#%EB%B3%B5%ED%95%A9%EC%9D%B8%EB%8D%B1%EC%8A%A4-%EC%84%A4%EA%B3%84-%EC%98%88%EC%A0%9C)


- 몽고는 결과가 32메가 이상이면 데이터가 너무 많아 정렬을 거부해버린다. -> limit을 써야함 

### 인덱스를 선택하는 방법?
- 5개 인덱스가 있다고 가정. 
- 쿼리가 들어오면 query shapde을 살펴본다 
- 충족하는 후보를 고른다 
- 3개가 식별됐다 
- 3개의 query plan을 만든다 
- 병렬 스레드에서 쿼리를 실행해서 젤 빠른애를 고른다 (캐싱된다, 시간이 지나거나 인덱스가 변경되면 제거됨)  
- 캐시는 삭제할 수 있고 재시작해도 사라진다. 

### 인덱스 주의점   
- 복합인덱스 정렬중요
- `커버드쿼리`를 실무에 잘 쓰자 
- 복합인덱스 활용은 아래처럼.
```
a b c d 복합인덱스라면
a ok
a b ok
b x
a b c ok
a c x
```
<br>
- $not $ne 인덱스 활용 어려움
- 복합인덱스 만들때 완전일치 필드를 앞에, 범위필드는 마지막에
- 하나의 쿼리에 두개의 범위 x
- 쿼리당 하나의 인덱스만 사용 but $or는 두개쓰고 병합함 & 중복제거함 -> $in 을 쓰는게 나음
<br>
- 내장 다큐와 배열도 인덱스가능하고 복합인덱스로 상위필드와 결합가능함
- 내장다큐 전체를 인덱싱하면 내장 다큐 전체를 쿼리할 때만 쓸수 있음
<br>
- 배열에도 인덱스를 생성할 수 있다. 
```
> db.blog.createIndex({"comments.date": 1})
```
> 배열필드를 인덱스 키로 가지면 인덱스는 즉시 다중키 인덱스로 표시된다 
> 한 게시물에 20개의 댓글이 달리면.. 도큐먼트는 20개의 인덱스 항목을 가지고.. CUD작업을 하면 모든 배열요소가 갱신되야 해서  
> 배열인덱스는 더 부담스럽다.  

<br>
---

### 인덱스 종류
- 고유인덱스
> 유니크해야 함  
> null도 취급하므로 nullable은 못만듬  
> `partialFilterExpression`    
- 복합고유인덱스
- 부분인덱스
<br>
---

### 인덱스 관리 
```
db.students.getIndexes()
db.people.dropIndex("x_1_y_1")
db.soup.createIndex({"a":1, "b":1, "c":-1})
```
> 인덱스 구축에는 시간이 오래걸리고 리소스가 많이 필요하다.  
> read, write가 중단될 수 있으므로 `background` 옵션을 사용해야 한다.  

<br>
---

### 공간정보 인덱스 
- `2dsphere`, `2d` 
```
> db.openStreetMap.createIndex({"loc": "2dsphere"})
> ... // insert
> db.openStreetMap.find({"loc": {"$geoWithin": {$geometry": eastVillage}}})
```
<br>
---

### 전문 검색을 위한 인덱스 
- text 인덱스 
> 제목, 설명 등 필드의 텍스트와 일치시키는 keyword query를 할때 사용  
> 정규표현식은 큰 텍스트 블록을 검색할 때 느리다  
> text 인덱스는 tokenization, stop word, stemming(형태소분석) 등 일반적인 검색 엔진 요구사항을 지원한다  
> text 인덱스에서 키의 개수는 필드의 단어 개수에 비례하므로 시스템 리소스가 많이 소비될 수 있다  
> 쓰기 성능이 떨어지고, 샤딩시 데이터 이동 속도가 느려진다.  
```
> db.articles.createIndex({"title":"text", "body":"text}, {"weights": {"title":3, "body":2}})
> db.articles.find({"$text": {"$search":"..."}}, {title:1}).limit(10)
```
<br>
---

### 제한 컬렉션 
- 유연성이 부족하지만 로깅에는 나름 유용하다 
- 환영큐처럼 동작.. 꽉 찼을 때 입력이 들어오면 가장 오래된 게 지워지고 그 자리에 입력된다.  
<br>
---

### TTL 인덱스 
```
> db.sessions.createIndex({"lastUpdated":1}, {"expireAfterSeconds": 60*60*24})
```
<br>
---

### GridFS로 파일 저장하기 
- 대용량 이진 파일을 저장하는 것 
- 대용량 파일을 청크로 나눈후, 각 청크를 도큐먼트로 저장할 수 있음
