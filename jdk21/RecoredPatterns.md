### Summary

**레코드 패턴**과 **타입 패턴**을 중첩해서 강력하고 **선언적**이며 **구성 가능한 형태**의 데이터 탐색 및 처리를 할 수 있다.

### History

JEP 405에서 preview feature로 제안됐고 JDK19에서 등장.

JEP 432에서 다시 제안됐고, JDK 20에 재등장.

JEP 440에서 완성되어 JDK 21에 탑재.

```java
# JEP 394

// Prior to Java 16
if (obj instanceof String) {
    String s = (String)obj;
    ... use s ...
}

// As of Java 16
if (obj instanceof String s) { 
    ... use s ...
}
```

> `instanceof` 연산자를 확장하여 **타입 패턴**을 취하고 패턴 매칭을 수행

```java
# JEP 395

// As of Java 16
record Point(int x, int y) {}

static void printSum(Object obj) {
    if (obj instanceof Point p) {
        int x = p.x();
        int y = p.y();
        System.out.println(x+y);
    }
}
```

> line 7: record 클래스와 타입 패턴의 활용
> 
> \*\* 레코드 클래스를 사용하면 간결한 방식으로 불변 데이터 객체 정의할 수 있다.

```java
// As of Java 21
static void printSum(Object obj) {
    if (obj instanceof Point(int x, int y)) {
        System.out.println(x+y);
    }
}
```

> line 3: `Point(int x, int y)` 가 레코드 패턴이다.
> 
> 추출하는 걸 deconstruct 라고 표현한다.
> 
> 인스턴스인지 여부를 검사할 뿐만 아니라 x, y의 값을 추출

```java
// As of Java 16
record Point(int x, int y) {}
enum Color { RED, GREEN, BLUE }
record ColoredPoint(Point p, Color c) {}
record Rectangle(ColoredPoint upperLeft, ColoredPoint lowerRight) {}
```

```java
// As of Java 21
static void printColorOfUpperLeftPoint(Rectangle r) {
    if (r instanceof Rectangle(ColoredPoint(Point p, Color c),
                               ColoredPoint lr)) {
        System.out.println(p);
        System.out.println(c);
        System.out.println(lr.p);
        System.out.println(lr.c);
    }
}
```

> 중첩 레코드 패턴을 통해 p, c, lr, lr.p, lr.c 를 추출할 수 있다.

```java
// As of Java 21
static void printUpperLeftColoredPoint(Rectangle r) {
    if (r instanceof Rectangle(ColoredPoint ul, ColoredPoint lr)) {
        System.out.println(ul.c());
        System.out.println(ul.p());
        System.out.println(lr.c());
        System.out.println(lr.p());
    }
}
```

> 중첩 레코드 패턴을 통해 ul, lr, ul.p, ul.c, lr.p, lr.c 를 추출할 수 있다.

```java
// As of Java 16
Rectangle r = new Rectangle(new ColoredPoint(new Point(x1, y1), c1), 
                            new ColoredPoint(new Point(x2, y2), c2));
                            
                            
// As of Java 21
static void printXCoordOfUpperLeftPointWithPatterns(Rectangle r) {
    if (r instanceof Rectangle(ColoredPoint(Point(var x, var y), var c),
                               var lr)) {
        System.out.println(x);
        System.out.println(y);
        System.out.println(c);
        System.out.println(lr);
        System.out.println(lr.p);
        System.out.println(lr.c);
        System.out.println(lr.p.x);
        System.out.println(lr.p.y);
    }
}
```

> `var`를 활용

```java
Pair p = new Pair(42, 42);

if (p instanceof Pair(String s, String t)) {
        System.out.println(s + ", " + t);
    } else {
        System.out.println("Not a pair of strings");
    }
```

> 실패하는 case

```java
record MyPair<S,T>(S fst, T snd){};
static void recordInference(MyPair<String, Integer> pair){
    switch (pair) {
        case MyPair(var f, var s) -> {
            System.out.println(f);
            System.out.println(s);
        }
    }
}
```

> type argument를 주지 않아도 된다