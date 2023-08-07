# 집계 

#### 2004년에 설립된 회사를 찾아보자 + 필드 몇개만
```
db.companies.aggregate([
  {$match: {founded_year: 2004}},
  {$sort: {name: 1}},
  {$skip: 10},
  {$limit: 5},
  {$project: {
    _id:0, name:1, founded_year: 1
  }}
])
```
> `aggregate` 는 집계 쿼리 실행시 호출하는 녀석이다.  
> 필터링을 위한 일치 단계  
> 출력을 위한 선출 단계  
> 제한 단계  
- 쿼리 플래너를 잘 생각해야 한다.  limit이 맨 밑에 있다면 선출 단계에서 꽤 많은 다큐먼트를 검색하게 된다.  
<br>
---

#### $project
> 선출 단계  
```
db.companies.aggregate([
  {$match: {"funding_rounds.investments.financial_org.permalink": "greylock"}},
  {$project: {
    _id: 0, name: 1,
    ipo: "$ipo.pub_year",
    valutation: "$ipo.valuation_amount",
    funders: "$funding_rounds.investments.financial_org.permalink" 
  }},
]).pretty()
``` 
> 그레이록 파트너스가 참여한 펀딩 라운드를 포함하는 모든 회사를 필터링한다  
> _id는 숨긴다, name은 포함한다  
> ipo와 funding_rounds 필드를 활용해서 선출 필드를 만든다  
<br>
---

### $unwind
> 배열 필드로 작업할 때 종종 하나 이상의 전계(unwind) 단계를 포함할 필요가 있다  
```
db.companies.aggregate([
  {$match: {"funding_rounds.investements.financial_org.permalink": "greylock}},
  {$unwind: "$funding_rounds"},
  {$project: .... }  
])
```
> funding_rounds 배열의 각 요소에 대한 출력 도큐먼트를 생성해서 집계한다  
<br>
---

### 배열 표현식 
```
db.companies.aggregate([
  {$match: {"funding_rounds.investements.financial_org.permalink": "greylock}},
  {$project:{
    _id: 0, name: 1, founded_year: 1, 
    rounds: {$filter: {
      input: "$funding_rounds",
      as: "round",
      cond: {$gte: ["$$round.raised_amount", 1000000]}
    }},
  }},
  {$match: {"rounds.investments.financial_org.permalink": "greylock"}}  
])
```
> rounds 필드는 필터 표현식을 사용  
> $filter 연산자는 배열 필드와 함께 작동하도록 설계됨.  
> $filter의 첫번째는 배열을 지정  
> $filter의 세번째는 조건, 조건 지정시 $$를 사용함. 표현식 내에 정의된 변수를 참조할 때 씀    
```
db.companies.aggregate([
  {$match: {"founded_year": 2010}},
  {$project:{
    _id: 0, name: 1, founded_year: 1, 
    first_round: {$arrayElemAt: ["$funding_rounds",0]},
    last_round: {$arrayElemAt: ["$funding_rounds",-1]},
  }}
])
```
> 선출 단계에서 $arrayElemAt이 사용됐다.  
<br>
---

### 그 외 
- 누산기
> $max, #min, $first, $last, $avg, $sum, $mergeObjects, $addToSet, $push  
- 그룹화 
>  $group  