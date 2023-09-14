*   읽기 어렵다
    

```java
String s = x + " plus " + y + " equals " + (x + y);
```

*   너무 장황하다
    

```java
String s = new StringBuilder()
                 .append(x)
                 .append(" plus ")
                 .append(y)
                 .append(" equals ")
                 .append(x + y)
                 .toString();
```

*   타입 불일치가 올 수 있다
    

```java
String s = String.format("%2$d plus %1$d equals %3$d", x, y, x + y);
String t = "%2$d plus %1$d equals %3$d".formatted(x, y, x + y);
```

*   수식이 많고 익숙하지 않은 구문
    

```java
MessageFormat mf = new MessageFormat("{0} plus {1} equals {2}");
String s = mf.format(x, y, x + y);
```

#### String interpolation

많은 언어들이 보간을 사용한다.

```java
C#             $"{x} plus {y} equals {x + y}"
Visual Basic   $"{x} plus {y} equals {x + y}"
Python         f"{x} plus {y} equals {x + y}"
Scala          s"$x plus $y equals ${x + y}"
Groovy         "$x plus $y equals ${x + y}"
Kotlin         "$x plus $y equals ${x + y}"
JavaScript     `${x} plus ${y} equals ${x + y}`
Ruby           "#{x} plus #{y} equals #{x + y}"
Swift          "\(x) plus \(y) equals \(x + y)"
```

자바에도?

#### interpolation 주의사항

보간은 주입 공격으로 이어질 수 있기 때문에 SQL 문에 특히 위험하다.

```java
String query = "SELECT * FROM Person p WHERE p.last_name = '${name}'";
ResultSet rs = connection.createStatement().executeQuery(query);
```

만약 name이 `Smith' OR p.last_name <> 'Smith` 라면 모든 행을 선택하게 된다.  
잠재적으로 기밀 정보를 노출할 것.

`SELECT * FROM Person p WHERE p.last_name = 'Smith' OR p.last_name <> 'Smith'`

그래서 단순하게 해서는 안된다. 약간의 편의성을 희생해서 큰 안전성을 얻어야 한다.

1급 템플릿 기반 메커니즘을 갖추면 거의 모든 java 프로그램의 가독성과 신뢰성을 향상 시킬 수 있다.

interpolation의 이점을 제공하면서 보안 취약점이 발생하는 경향도 줄인다.

*   STR
    

```java
String name = "Joan";
String info = STR."My name is \{name}";
assert info.equals("My name is Joan");   // true

// Embedded expressions can be strings
String firstName = "Bill";
String lastName  = "Duck";
String fullName  = STR."\{firstName} \{lastName}";
| "Bill Duck"
String sortName  = STR."\{lastName}, \{firstName}";
| "Duck, Bill"

// Embedded expressions can perform arithmetic
int x = 10, y = 20;
String s = STR."\{x} + \{y} = \{x + y}";
| "10 + 20 = 30"

// Embedded expressions can invoke methods and access fields
String s = STR."You have a \{getOfferType()} waiting for you!";
| "You have a gift waiting for you!"
String t = STR."Access at \{req.date} \{req.time} from \{req.ipAddress}";
| "Access at 2022-03-25 15:34 from 8.8.8.8"
```

*   old vs new
    

```java
String filePath = "tmp.dat";
File   file     = new File(filePath);
String old = "The file " + filePath + " " + (file.exists() ? "does" : "does not") + " exist";
String msg = STR."The file \{filePath} \{file.exists() ? "does" : "does not"} exist";
| "The file tmp.dat does exist" or "The file tmp.dat does not exist"
```

*   To aid readability
    

```java
String time = STR."The time is \{
    // The java.time.format package is very useful
    DateTimeFormatter
      .ofPattern("HH:mm:ss")
      .format(LocalTime.now())
} right now";
| "The time is 12:34:56 right now"
```

*   array
    

```java
String s = STR."\{fruit[0]}, \{
    STR."\{fruit[1]}, \{fruit[2]}"
}";
```

*   multi-line
    

```java
String title = "My Web Page";
String text  = "Hello, world";
String html = STR."""
        <html>
          <head>
            <title>\{title}</title>
          </head>
          <body>
            <p>\{text}</p>
          </body>
        </html>
        """;

String name    = "Joan Smith";
String phone   = "555-123-4567";
String address = "1 Maple Drive, Anytown";
String json = """
    {
        "name":    "%s",
        "phone":   "%s",
        "address": "%s"
    }
    """.formatted(name, phone, address);

String table = STR."""
    Description  Width  Height  Area
    \{zone[0].name}  \{zone[0].width}  \{zone[0].height}     \{zone[0].area()}
    \{zone[1].name}  \{zone[1].width}  \{zone[1].height}     \{zone[1].area()}
    \{zone[2].name}  \{zone[2].width}  \{zone[2].height}     \{zone[2].area()}
    Total \{zone[0].area() + zone[1].area() + zone[2].area()}
    """;
```

*   FMT
    

```java
record Rectangle(String name, double width, double height) {
    double area() {
        return width * height;
    }
}
Rectangle[] zone = new Rectangle[] {
    new Rectangle("Alfa", 17.8, 31.4),
    new Rectangle("Bravo", 9.6, 12.4),
    new Rectangle("Charlie", 7.1, 11.23),
};
String table = FMT."""
    Description     Width    Height     Area
    %-12s\{zone[0].name}  %7.2f\{zone[0].width}  %7.2f\{zone[0].height}     %7.2f\{zone[0].area()}
    %-12s\{zone[1].name}  %7.2f\{zone[1].width}  %7.2f\{zone[1].height}     %7.2f\{zone[1].area()}
    %-12s\{zone[2].name}  %7.2f\{zone[2].width}  %7.2f\{zone[2].height}     %7.2f\{zone[2].area()}
    \{" ".repeat(28)} Total %7.2f\{zone[0].area() + zone[1].area() + zone[2].area()}
    """;
| """
| Description     Width    Height     Area
| Alfa            17.80    31.40      558.92
| Bravo            9.60    12.40      119.04
| Charlie          7.10    11.23       79.73
|                              Total  757.69
| """
```

*   error
    

```java
String name = "Joan";
String info = "My name is \{name}";
| error: processor missing from template expression
```

*   RAW
    

```java
int y = 20;
for (int x = 0; x < 3; x++) {
    StringTemplate st = RAW."\{x} plus \{y} equals \{x + y}";
    System.out.println(st);
}
| ["Adding ", " and ", " yields ", ""](0, 20, 20)
| ["Adding ", " and ", " yields ", ""](1, 20, 21)
| ["Adding ", " and ", " yields ", ""](2, 20, 22)

```