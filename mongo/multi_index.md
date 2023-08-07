# 몽고디비 인덱스 

## 복합인덱스 설계 예제 

스캔할 레코드 개수를 인덱스가 얼마나 최소화 하느냐가 중요함.   
>  동등 필터에 대한 키를 맨 앞에 표시해야 한다.   
>  정렬에 사용되는 키는 다중값 필드 앞에 표시해야 한다   
>  다중값 필터에 대한 키는 마지막에 표시해야 한다   
---

- 모델 
```
{
  "_id": ObjectId("...),
  "student_id": 0, 
  "scores" : [
    {
      "type": "exam",
      "score": 38.05283028
    },
    ...
  ],
  "class_id": 127
}
...
```

- 생성된 인덱스
```
db.students.createIndex({"class_id": 1})
db.students.createIndex({student_id: 1, class_id: 1})
```

- 실행할 쿼리 
```
db.students.find({student_id:{$gt:500000}, class_id:54})
  .sort({stuends_id:1})
  .explain("executionStats")
```

- 실행계획 결과
```
{
  "queryPlanner": {
    ...
  },
  "winningPlan": {
    stage: FETCH,
    indexName: student_id_1_class_id_1, // 선정된 녀석
    ...
  },
  "rejectedPlans": [
    {
      stage: SORT, // <-- 요것도 문제
      sortPattern: {
        student_id: 1                    
      },                               
      ...
        indexName: class_id_1,
      ...
    }
  ],
  "executionStats": {
    nReturned: 9903,            // 리턴한 다큐먼트 수
    executionTimeMillis: 4325,  // 실행 시간
    totalKeysExamined: 850477,  // 찾아본 인덱스 키 수 <-- 요것도 문제
    totalDocsExamined: 9903,    // 찾아본 다큐먼트 수 
    ...
  }
  ...
}
```
- - 찾아본 인덱스 키 수가 엄청 많다. 
- - 복합인덱스 student_id_1_class_id_1가 사용됐다 
- - rejectedPlans를 보면 class_1은 사용되지 않았고, SORT가 여기 있는걸 보니 인메모리 정렬을 했음을 알 수 있다. 
- - class_id: 54를 써서 제한했음에도 불구하고 정렬은 student_id를 가지고 하다보니 85만개를 스캔하게 됐다. 
- - 우리가 가진 인덱스를 쓰도록 강제하는 방법이 있다. 
```
# hint 
db.students.find({student_id: {$gt: 500000}, class_id:54})
  .sort({student_id:1})
  .hint({class_id:1}) // <---
  .explain("executionStats")

winningPlan
  stage: SORT // <-- 요것도 문제
    indexName: class_id_1

executionTimeMillis: 272
totalKeysExamined: 20076
totalDocsExamined: 20076
```
- - 훨씬 빨라졌다. 
- - class_id_1는 사용하게 됐지만 student_id_1_class_id_1가 사용하는게 없어졌다 
- - nReturned과 totalKeysExamined를 유사하게 만들고 싶다 + hintf를 사용하고 싶지 않다면?
```
// db.students.createIndex({student_id: 1, class_id: 1})
db.students.createIndex({class_id: 1, student_id: 1})  // <-- 요걸로 변경! 

stage: IXSCAN
executionTimeMillis: 37
totalKeysExamined: 9903
totalDocsExamined: 9903
indexName: student_id_1_class_id_1
```
- - 결국 복합인덱스의 순서가 중요하다. class는 student보다 대분류 이므로 앞에 오는게 좋다는 것. 
- - 만약 sort는 final_grade로 한다면?
```
stage: SORT  // <-- 요것도 문제
stage: IXSCAN
executionTimeMillis: 136
totalKeysExamined: 9903
totalDocsExamined: 9903
indexName: student_id_1_class_id_1
```
- - 다소 느려졌다. SORT가 있는걸로봐서 인메모리 정렬을 했다. (인덱스가 없으니깐ㅋ)
```
db.students.createIndex({class_id: 1, final_grade:1, student_id: 1})  // <-- 요거로 변경! 
```
- - 이렇게 하면 인메모리 정렬없이 빠른 실행결과를 받아볼 수 있다. 
- - 다만 final_grade까지 해야 하는가에 대해서는 고찰을 해봐야 할 듯. 
