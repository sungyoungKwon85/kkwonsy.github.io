몽고를 사용하는 이유 
  분산확장 
  도큐먼트를 사용함으로써 복잡한 계층 관계를 하나의 레코드로 표현할 수 있다 
  고정된 스키마가 아니므로 쉽게 필드를 추가/삭제 할 수 있다 

기본

  모든 도큐먼트는 16메가바이트보다 작아야 한다 (소설 한권 넘음..)

  db.movies.insertOne({...})
  db.movies.insertMany([{...}])
  insert는 쓰지말자

  db.movies.findOne().pretty()

  db.movies.updateOne({title: "Star Wars2"}, {"$set": {reviews: []}}) // $Set은 필드값을 설정한다, 값만 변경도 되고, 데이터형도 바꿀 수 있다. 
  db.movies.updateMany
  db.movies.updateOne({title: "Star Wars2"}, {"$unset": {favorite book: 1}}) // $unset은 키와 값을 제거한다. 
  db.movies.updateOne({title: "Star Wars2"}, {"$inc": {score: 50}}) // $inc는 값을 더해준다, 숫자형에만 사용 가능  
  db.movies.updateOne({title: "Star Wars2"}, {"$push": {
    "comments": {
      "name": "joe", 
      "content": "nice post",
      "email": "joe@example.com"
    }
  }}) // $push는 배열이 존재하면 끝에 넣고, 존재하지 않으면 새로 생성해준다 
  db.stock.ticker.updateOne({"_id": "GOOG"}, { "$push": {"hourly": { "$each": [1.1, 1.2, 1.3]}}}) // $each를 쓰면 배열에 여러번 추가된다 
  
  $addToSet // 중복을 피할 수 있다 
  $pop // 배열을 stack, queue처럼 요소 제거할 때 쓴다
  $pull // 조건에 맞는 요소를 제거한다 

  db.blog.updateOne({"comments.author": "John"}, {"$set": {"comments.$.author": "Jim"}}) // $ 는 위치연산자이다. 첫번째로 일치하는 요소만 갱신한다. 

  $setOnInsert // 도큐먼트 생성시 필드가 설정돼야 할 때 사용 

  // 갱신된 도큐먼트를 반환 
     updateOne과 차이점은.. 수정된 도큐먼트의 값을 원자적으로 얻을 수 있다는 것. 
  findAndModify // 삭제, 대체, 갱신 기능이 결합돼 사용자 오류가 발생하기 쉽다 
  findOneAndDelete
  findOneAndReplace
  findOneAndUpdate

  $in, $nin
  $gte $ne $lte $lt $gt
  $or 
  $not
  $mod  // 나머지 
  $exists
  "$regex": /joe/i // 대소문자 구별없이 찾기 
  "$regex": /joey?/i // joe, joey등을 찾고 싶을 때 
  db.food.find({fruit: {$all: ["apple", "banana"]}}) // 배열에서 2개이상 일치하는 것을 찾을 때 
  $size
  db.blog.posts.findOne(criteria, {"comments": {"$slice": 10}}) // 배열요소의 부분집합을 받을 수 있다. 
  $where // javascript를 사용해서 필터를 건다. 보안상 이유로 사용잘하려나?

  var cursor = db.people.find({}) // 커서는 이런식으로 쓸 수 있음, 커서는 리소스를 잡고 있으므로 조심해서 써야 함  
  cursor.forEach(function(x){ print(x.name);});

  .limit(3)
  .skip(10) // 생략된 결과물을 찾은 뒤 폐기하므로 성능이슈 있음. 
  .sort({username: 1, age: -1})
  
  db.people.replaceOne({"_id": ObjectId("...")}, joe) // 치환  

  db.movies.deleteOne({title: "Star Wars2"})
  db.movies.deleteMany
  remove는 쓰지말자 

  

데이터형
  {"x": NumberInt("3")} // 4, 8바이트 부호 정수
  {"x": NumberLong("3")}

  {"x": new Date()} // 64bit정수, time zone 없음 

  {"x": /foobar/i} // 정규표현식 

  {"x": ObjectId()} // 객체 ID 

  {"x": function() { /* ... */ }} // code 

내장 도큐먼트 
  도큐먼트는 키에 대한 값이 될 수 있다 
  {
    "name": "kwon",
    "address": { 
      "street": "123",
      "city": "seoul"
    }
  }
  // RDB는 두개의 테이블로 모델링 됨 
  // 다만 data repition이 발생할 수 있다는 단점. 

ObjectId()
  _id의 기본 데이터형 
  분산확장에 문재없도록 설계되었다 

---

셸 활용
  $ mongo script1.js script2.js sript4.js // 자바스크립트 파일을 셸로 전달할 수 있음 

.mognorc.js
  를 만들면 셸이 시작할 때 마다 필요한 부분을 실행시킬 수 있다 
  db.dropDatabase = DB.prototype.dropDatabase = no; // 데이터베이스 삭제를 방지 시킴 
  
---

인덱싱 
  인덱스를 사용하지 않는 쿼리를 컬렉션 스캔이라 한다.
  성능을 위해 인덱스를 사용해야 한다 
    단점? CUD작업은 인덱스를 수정해야 하기 때문에 READ보다 CUD가 훨씬 많다면 다시 생각해봐야함. 

  .explain("executionStats")을 사용하면 성능을 확인해볼 수 있다. 
    살펴보면 좋은 필드. 
    nReturned
    totalDocsExamined
    executionTimeMillis 

  어떤필드? 
    자주쓰는 쿼리를 조사해 공통적인 키 셋을 찾아볼 것. 

  복합인덱스?
    상당 수 쿼리 패턴은 두 개 이상의 키를 기반으로 인덱스를 작성한다 
    db.users.find().sort({"age":1, "username": 1}) 이 많이 사용된다면 
    age와 username을 이용해 인덱스를 만든다. 
    순서가 중요하다. 
    db.users.createIndex({"age":1, "username": 1})

    스캔할 레코드 개수를 인덱스가 얼마나 최소화 하느냐가 중요함. 
      동등 필터에 대한 키를 맨 앞에 표시해야 한다. 
      정렬에 사용되는 키는 다중값 필드 앞에 표시해야 한다 
      다중값 필터에 대한 키는 마지막에 표시해야 한다 
      *인덱스는 중요해서 따로 문서 저장! 



  몽고는 결과가 32메가 이상이면 데이터가 너무 많아 정렬을 거부해버린다. -> limit을 써야함 

  인덱스를 선택하는 방법?
    5개 인덱스가 있다고 가정. 
    쿼리가 들어오면 query shapde을 살펴본다 
    충족하는 후보를 고른다 
    3개가 식별됐다 
    3개의 query plan을 만든다 
    병렬 스레드에서 쿼리를 실행해서 젤 빠른애를 고른다 (캐싱된다, 시간이 지나거나 인덱스가 변경되면 제거됨)  
    캐시는 삭제할 수 있고 재시작해도 사라진다. 

  


  



