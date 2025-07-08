# 템플릿

템플릿이란 바뀌는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법이다.

## 변하는 것과 변하지 않는 것
### 분리와 재사용을 위한 디자인 패턴 적용
변하는 부분을 변하지 않는 나머지 코드에서 분리하는 것이 어떨까?

그렇게 할 수 있다면 변하지 않는 부분을 재사용할 수 있는 방법이 있지 않을까?

먼저 생각해볼 수 있는 방법은 변하는 부분을 메소드로 빼는 것이다.

```java
public void deleteAll() throws SQLException {
  ...
  try {
    c = dataSource.getConnection();

    ps = makeStatement(c); // 변하는 부분을 메소드로 추출하고 변하지 않는 부분에서 호출하도록 만들었다.
    ps.executeUpdate();
  } catch (SQLException e)
  ...
}

private PreparedStatement makeStatement(Connection c) throws SQLException {
  PreparedStatement ps;
  ps = c.prepareStatement("delete from users");
  return ps;
}
```

자주 바뀌는 부분을 메소드로 독립시켰는데 당장 봐서는 별 이득이 없어 보인다.

왜냐하면 보통 메소드 추출 리팩토링을 적용하는 경우에는 분리시킨 메소드를 다른 곳에서 재사용할 수 있어야 하는데, 이건 반대로 분리시키고 남은 메소드가 재사용이 필요한 부분이고, 분리된 메소드는 DAO 로직마다 새롭게 만들어서 확장돼야 하는 부분이기 때문이다.

템플릿 메소드 패턴은 상속을 통해 기능을 확장해서 사용하는 부분이다.

변하지 않는 부분은 슈퍼클래스에 두고 변하는 부분은 추상 메소드로 정의해둬서 서브클래스에서 오버라이드하여 새롭게 정의해 쓰도록 하는 것이다.

makeStatement() 메소드를 다음과 같이 추상 메소드 선언으로 변경한다.

```java
abstract protected PreparedStatement makeStatement(Connection c) throws SQLException;
```

메소드와 필요에 따라서 상속을 통해 구체적인 PreparedStatement를 바꿔서 사용할 수 있게 만드는 서브클래스로 깔끔하게 분리할 수 있다.

```java
public class UserDaoDeleteAll extends UserDao {
  protected PreparedStatement makeStatement(Connection c) throws SQLException {
    PreparedStatement ps = c.prepareStatement("delete from users");
    return ps;
  }
}
```

이제 UserDao 클래스의 기능을 확장하고 싶을 때마다 상속을 통해 자유롭게 확장할 수 있고, 확장 때문에 기존의 상위 DAO 클래스에 불필요한 변화는 생기지 않도록 할 수 있으니 객체지향 설계의 핵심 원리인 개방 폐쇄 원칙을 그럭저럭 지키는 구조를 만들어낼 수는 있는 것 같다.

![image](https://github.com/user-attachments/assets/4e428adf-f7c8-4f9c-839d-ea6e03740be9)

확장구조가 이미 클래스를 설계하는 시점에서 고정되어 버린다는 점과 관계에 대한 유연성이 떨어져버리는 점이 상속을 통해 확장을 꾀하는 템플릿 메소드 패턴의 단점으로 드러난다.

개방 폐쇄 원칙을 잘 지키는 구조이면서도 템플릿 메소드 패턴보다 유연하고 확장성이 뛰어난 것이, 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 전략 패턴이다.

전략 패턴은 OCP 관점에 보면 확장에 해당하는 변하는 부분을 별도의 클래스로 만들어 추상화된 인터페이스를 통해 위임하는 방식이다.

![image](https://github.com/user-attachments/assets/cf667393-fe62-425d-a8f4-b926e6a7a64e)

좌측에 있는 Context의 contextMethod()에서 일정한 구조를 가지고 동작하다가 특정 확장 기능은 Strategy 인터페이스를 통해 외부의 독립된 전략 클래스를 위임하는 것이다.

deleteAll() 메소드에서 변하지 않는 부분이라고 명시한 것이 바로 이 contextMethod()가 된다.

deleteAll()은 JDBC를 이용해 DB를 업데이트하는 작업이라는 변하지 않는 맥락을 갖는다.

deleteAll()의 컨텍스트를 정리해보면 다음과 같다.
- DB 커넥션 가져오기
- PreparedStatement를 만들어줄 외부 기능 호출하기
- 전달받은 PreparedStatement 실행하기
- 예외가 발생하면 이를 다시 메소드 밖으로 던지기
- 모든 경우에 만들어진 PreparedStatement와 Connection을 적절히 닫아주기

두 번째 작업에서 사용하는 PreparedStatement를 만들어주는 외부 기능이 바로 전략 패턴에서 말하는 전략이라고 볼 수 있다.

전략 패턴의 구조를 따라 이 기능을 인터페이스로 만들어두고 인터페이스의 메소드를 통해 PreparedStatement 생성 전략을 호출해주면 된다.

```java
public interface StatementStrategy {
  PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}
```

이 인터페이스를 상속해서 실제 전략, 즉 바뀌는 부분인 PreparedStatement를 생성하는 클래스를 만든다.

```java
public class DeleteAllStatement implements StatementStrategy {
  public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
    PreparedStatement ps = c.prepareStatement("delete from users");
    return ps;
  }
}
```

```java
public void deleteAll() throws SQLException {
  ...
  try {
    c = dataSource.getConnection();

    StatementStrategy strategy = new DeleteAllStatement();
    ps = strategy.makePreparedStatement(c);
    
    ps.executeUpdate();
  } catch (SQLException e)
  ...
}
```

전략 패턴은 필요에 따라 컨텍스트는 그대로 유지되면서(OCP의 폐쇄 원칙) 전략을 바꿔 쓸 수 있다(OCP의 개방 원칙)는 것인데, 이렇게 컨텍스트 안에서 이미 구체적인 전략 클래스인 DeleteAllStatement를 사용하도록 고정되어 있다면 뭔가 이상하다.

컨텍스트가 StatementStrategy 인터페이스뿐 아니라 특정 구현 클래스인 DeleteAllStatement를 직접 알고 있다는 건, 전략 패턴에도 OCP에도 잘 들어맞는다고 볼 수 없기 때문이다.

전략 패턴에 따르면 Context가 어떤 전략을 사용할 것인가는 Context를 사용하는 앞단의 Client가 결정하는게 일반적이다.

Client가 구체적인 전략의 하나를 선택하고 오브젝트로 만들어서 Context에 전달하는 것이다.

Context는 전달받은 그 Strategy 구현 클래스의 오브젝트를 사용한다.

![image](https://github.com/user-attachments/assets/1363c1b8-d4a7-4973-9d26-9c5871a5c057)

```java
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException { // stmt => 클라이언트가 컨텍스트를 호출할 때 넘겨줄 전략 파라미터
  Connection c = null;
  PreparedStatement ps = null;

  try {
    c = dataSource.getConnection();

    ps = stmt.makePreparedStatement(c);

    ps.executeUpdate();
  } catch(SQLException e) {
    throw e;
  } finally {
    if (ps != null) ...
    if (c != null) ...
  }
}
```

컨텍스트를 별도의 메소드로 분리했으니 deleteAll() 메소드가 클라이언트가 된다.

deleteAll()은 전략 오브젝트를 만들고 컨텍스트를 호출하는 책임을 지고 있다.

사용할 전략 클래스는 DeleteAllStatement이므로 이 클래스의 오브젝트를 생성하고, 컨텍스트로 분리한 jdbcContextWithStatementStrategy() 메소드를 호출해주면 된다.

```java
public void deleteAll() throws SQLException {
  StatementStrategy st = new DeleteAllStatement(); // 선정한 전략 클래스의 오브젝트 생성
  jdbcContextWithStatementStrategy(st); // 컨텍스트 호출, 전략 오브젝트 전달
}
```

## JDBC 전략 패턴의 최적화
### 전략 클래스의 추가 정보
add() 메소드에도 적용한다.

먼저 add() 메소드에서 변하는 부분인 PreparedStatement를 만드는 코드를 AddStatement 클래스로 옮겨 담는다.

```java
public class AddStatement implements StatementStrategy {
  public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
    PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?, ?, ?)");
    ps.setString(1, user.getId());
    ps.setString(2, user.getName());
    ps.setString(3, user.getPassword()); // 그런데 user는 어디서 가져올까?

    return ps;
  }
}
```
클래스를 분리하고 나니 컴파일 에러가 발생한다.

deleteAll()과는 달리 add()에서는 PreparedStatement를 만들 때 user라는 부가적인 정보가 필요하기 때문이다.

등록할 사용자 정보는 클라이언트에 해당하는 add() 메소드가 갖고 있다.

따라서 클라이언트가 AddStatement의 전략을 수행하려면 부가정보인 user를 제공해줘야 한다.

AddStatement의 생성자를 통해 제공받게 만들자.

```java
public class AddStatement implements StatementStrategy {
  User user;

  public AddStatement(User user) {
    this.user = user;
  }

  ...
}
```

클라이언트인 UserDao의 add() 메소드를 user 정보를 생성자를 통해 전달해주도록 수정한다.

```java
public void add(User user) throws SQLException {
  StatementStrategy st = new AddStatement(user);
  jdbcContextStatementStrategy(st);
}
```

### 전략과 클라이언트의 동거
깔끔하게 코드를 만들었지만, 2가지 개선사항이 있다.

먼저, DAO 메소드마다 새로운 StatementStrategy 구현 클래스를 만들어야 한다는 점이다.

이렇게 되면 기존보다 클래스 파일의 개수가 많이 늘어난다.

또 다른 점은 DAO 메소드에서 StatementStrategy에 전달할 User와 같은 부가적인 정보가 있는 경우, 이를 위해 오브젝트를 전달받는 생성자와 이를 저장해둘 인스턴스 변수를 번거롭게 만ㄷ르어야 한다는 점이다.

이 오브젝트가 사용되는 시점은 컨텍스트가 전략 오브젝트를 호출할 때이므로 잠시라도 어딘가에 다시 저장해둘 수밖에 없다.

클래스 파일이 많아지는 문제는 간단한 해결 방법이 있다.

StatementStrategy 전략 클래스를 매번 독립된 파일로 만들지 말고 UserDao 클래스 안에 내부 클래스로 정의해버리는 것이다.

DeleteAllStatement나 AddStatement는 UserDao 밖에서는 사용되지 않는다.

둘 다 UserDao에서만 사용되고, UserDao의 메소드 로직에 강하게 결합되어 있다.

```java
public void add(User user) throws SQLException {
  class AddStatement implements StatementStrategy {
    User user;

    public AddStatement(User user) {
      this.user = user;
    }

    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
      PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?, ?, ?)");
      ps.setString(1, user.getId());
      ps.setString(2, user.getName());
      ps.setString(3, user.getPassword());

      return ps;
    }
  }
}
```

#### 중첩 클래스의 종류
```
  다른 클래스 내부에 정의되는 클래스를 중첩 클래스라고 한다.
  
  중첩 클래스는 독립적으로 오브젝트로 만들어질 수 있는 스태틱 클래스(static class)와 자신이 정의된 클래스의 오브젝트 안에서만 만들어질 수 있는 내부 클래스(inner class)로 구분된다.
  
  내부 클래스는 다시 범위에 따라 세 가지로 구분된다.
  
  멤버 필드처럼 오브젝트 레벨에 정의되는 멤버 내부 클래스와 메소드 레벨에 정의되는 로컬 클래스, 그리고 이름을 갖지 않는 익명 내부 클래스다.
  
  익명 내부 클래스의 범위는 선언된 위치에 따라 다르다.
```

AddStatement 클래스를 로컬 클래스로서 add() 메소드 안에 집어넣은 것이다.

로컬 클래스는 선언된 메소드 내에서만 사용할 수 있다.

AddStatement가 사용될 곳이 add() 메소드뿐이라면, 이렇게 사용하기 전에 바로 정의해서 쓰는 것도 나쁘지 않다.

덕분에 클래스 파일이 하나 줄었고, add() 메소드 안에서 PreparedStatement 생성 로직을 함께 볼 수 있으니 코드를 이해하기도 좋다.

로컬 클래스에는 또 한 가지 장점이 있다.

바로 로컬 클래스는 클래스가 내부 클래스이기 때문에 자신이 선언된 곳의 정보에 접근할 수 있다는 점이다.

내부 클래스에서 외부의 변수를 사용할 때는 외부 변수는 반드시 final로 선언해줘야 한다.

user 파라미터는 메소드 내부에서 변경될 일이 없으므로 final로 선언해도 무방하다.

이렇게 내부 클래스의 장점을 이용하면 user 정보를 전달받기 위해 만드렁ㅆ던 생성자와 인스턴스 변수를 제거할 수 있으므로 AddStatement는 간결해진다.

```java
public void add(final User user) thorws SQLException {
  class AddStatement implements StatementStrategy {
    ...
  }

  StatementStrategy st = new AddStatement(); // 생성자 파라미터로 user를 전달하지 않아도 된다.
  jdbcContextWithStatementStrategy(st);
}
```

AddStatement 클래스는 add() 메소드에서만 사용할 용도로 만들어졌다.

그렇다면 좀 더 간결하게 클래스 이름도 제거할 수 있다.

#### 익명 내부 클래스
```
  익명 내부 클래스는 이름을 갖지 않는 클래스다.
  
  클래스 선언과 오브젝트 생성이 결합된 형태로 만들어지며, 상속할 클래스나 구현할 인터페이스를 생성자 대신 사용한다.
  
  클래스를 재사용할 필요가 없고, 구현한 인터페이스 타입으로만 사용할 경우에 유용하다.
  
  new 인터페이스이름() { 클래스 본문 };
```

익명 내부 클래스는 선언과 동시에 오브젝트를 생성한다.

이름이 없기 때문에 클래스 자신의 타입을 가질 수 없고, 구현한 인터페이스 타입의 변수에만 저장할 수 있다.

```java
StatementStrategy st = new StatementStrategy() { // 익명 내부 클래스는 구현하는 인터페이스를 생성자처럼 이용해서 오브젝트로 만든다.
  public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
    ...
  }
}
```

## 컨텍스트와 DI
### JdbcContext의 특별한 DI
지금까지 적용했던 DI에서는 클래스 레벨에서 구체적인 의존관계가 만들어지지 않도록 인터페이스를 사용했다.

인터페이스를 적용했기 때문에 코드에서 직접 클래스를 사용하지 않아도 됐고, 그 덕분에 설정을 변경하는 것만으로도 얼마든지 다양한 의존 오브젝트를 변경해서 사용할 수 있게 됐다.

인터페이스를 사용하지 않고 DI를 적용하는 것은 문제가 있지 않을까?

상관은 없다. 하지만 꼭 그럴 필요는 없다.

의존관계 주입(DI) 개념을 충실히 따르자면, 인터페이스를 사이에 둬서 클래스 레벨에서는 의존관계가 고정되지 않게 하고, 런타임 시에 의존할 오브젝트와의 관계를 다이나믹하게 주입해주는 것이 맞다.

따라서 인터페이스를 사용하지 않았다면 엄밀히 말해서 온전한 DI라고 볼 수는 없다.

그러나 스프링의 DI는 넓게 보자면 객체의 생성과 관계설정에 대한 제어권한을 오브젝트에서 제거하고 외부로 위임했다는 IoC라는 개념을 포괄한다.

DI를 위해서는 주입되는 오브젝트와 주입받는 오브젝트 양쪽 모두 스프링 빈으로 등록돼야 한다.

중요한 것은 인터페이스의 사용 여부다.

왜 인터페이스를 사용하지 않았을까?

인터페이스가 없다는 것은 매우 긴밀한 관계를 가지고 강하게 결합되어 있다는 의미다.

## 템플릿과 콜백
복잡하지만 바뀌지 않는 일정한 패턴을 갖는 작업 흐름이 존재하고 그 중 일부분만 자주 바꿔서 사용해야 하는 경우에 적합한 구조다.

전략 패턴의 기본 구조에 익명 내부 클래스를 활용한 방식이다.

이런 방식을 스프링에서는 "템플릿/콜백 패턴"이라고 부른다.

전략 패턴의 컨텍스트를 템플릿이라 부르고, 익명 내부 클래스로 만들어지는 오브젝트를 콜백이라고 부른다.

#### 템플릿
```
  템플릿(template)은 어떤 목적을 위해 미리 만들어둔 모양이 있는 틀을 가리킨다.
  
  템플릿 메소드 패턴은 고정된 틀의 로직을 가진 템플릿 메소드를 슈퍼클래스에 두고, 바뀌는 부분을 서브클래스의 메소드에 두는 구조로 이뤄진다.
```

#### 콜백
```
  콜백은 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달되는 오브젝트를 말한다.
  
  파라미터로 전달되지만 값을 참조하기 위한 것이 아니라 특정 로직을 담은 메소드를 실행시키기 위해 사용한다.
  
  자바에선 메소드 자체를 파라미터로 전달할 방법이 없기 때문에 메소드가 담긴 오브젝트를 전달해야 한다.
```

### 동작원리
템플릿은 고정된 작업 흐름을 가진 코드를 재사용한다는 의미에서 붙인 이름이다.

콜백은 템플릿 안에서 호출되는 것을 목적으로 만들어진 오브젝트를 말한다.

여러 개의 메소드를 가진 일반적인 인터페이스를 사용할 수 있는 전략 패턴의 전략과 달리 템플릿/콜백 패턴의 콜백은 보통 단일 메소드 인터페이스를 사용한다.

템플릿의 작업 흐름 중 특정 기능을 위해 한 번 호출되는 경우가 일반적이기 때문이다.

하나의 템플릿에서 여러 가지 종류의 전략을 사용해야 한다면 하나 이상의 콜백 오브젝트를 사용할 수도 있다.

콜백은 일반적으로 하나의 메소드를 가진 인터페이스를 구현한 익명 내부 클래스로 만들어진다고 보면 된다.

콜백 인터페이스의 메소드에는 보통 파라미터가 있다.

이 파라미터는 템플릿의 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달받을 때 사용된다.

![image](https://github.com/user-attachments/assets/427d8ab4-b2b7-4cec-9b64-4757bc8cb965)

- 클라이언트의 역할은 템플릿 안에서 실행될 로직을 담은 콜백 오브젝트를 만들고, 콜백이 참조할 정보를 제공하는 것이다. </br>
  만들어진 콜백은 클라이언트가 텀플릿의 메소드를 호출할 때 파라미터로 전달된다.
- 템플릿은 정해진 작업 흐름에 따라 작업을 진행하다가 내부에서 생성한 참조정보를 가지고 콜백 오브젝트의 메소드를 호출한다. </br> 콜백은 클라이언트 메소드에 있는 정보와 템플릿이 제공한 참조정보를 이용해서 작업을 수행하고 그 결과를 다시 템플릿에 돌려준다.
- 템플릿은 콜백이 돌려준 정보를 사용해서 작업을 마저 수행한다. </br> 경우에 따라 최종 결과를 클라이언트에 다시 돌려주기도 한다.

클라이언트가 템플릿 메소드를 호출하면서 콜백 오브젝트를 전달하는 것은 메소드 레벨에서 일어나는 DI다.

일반적인 DI라면 템플릿에 인스턴스 변수를 만들어두고 사용할 의존 오브젝트를 수정자 메소드로 받아서 사용할 것이다.

반면에 템플릿/콜백 방식에서는 매번 메소드 단위로 사용할 오브젝트를 새롭게 전달받는다는 것이 특징이다.

콜백 오브젝트가 내부 클래스로서 자신을 생성한 클라이언트 메소드 내의 정보를 직접 참조한다는 것도 템플릿/콜백의 고유한 특징이다.

클라이언트와 콜백이 강하게 결합된다는 면에서도 DI와 조금 다르다.

### 편리한 콜백의 재활용
템플릿/콜백 방식에서 한 가지 아쉬운 점이 있다.

DAO 메소드에서 매번 익명 내부 클래스를 사용하기 때문에 상대적으로 코드를 작성하고 읽기가 조금 불편하다는 점이다.

```java
public void deleteAll() throws SQLException {
  executeSql("delete from users");
}

private void executeSql(final String query) throws SQLException {
  this.jdbcContext.workWithStatementStrategy (new StatementStrategy() {
    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
      return c.prepareStatement(query);
    }
  }
}
```
바뀌지 않는 모든 부분을 빼내서 executeSql() 메소드로 만들었다.

SQL을 담은 파라미터를 final로 선언해서 익명 내부 클래스인 콜백 안에서 직접 사용할 수 있게 하는 것만 주의하면 된다.

변하는 것과 변하지 않는 것을 분리하고 변하지 않는 건 유연하게 재활용할 수 있게 만든다는 간단한 원리를 계속 적용했을 때 이렇게 단순하면서도 안전하게 작성 가능한 코드가 완성된다.

### 템플릿/콜백의 응용

고정된 작업 흐름을 갖고 있으면서 여기저기서 자주 반복되는 코드가 있다면, 중복되는 코드를 분리할 방법을 생각해보는 습관을 기르자.

중복된 코드는 먼저 메소드로 분리하는 간단한 시도를 해본다.

그중 일부 작업을 필요에 따라 바꾸어 사용해야 한다면 인터페이스를 사이에 두고 분리해서 전략 패턴을 적용하고 DI로 의존관계를 관리하게 만든다.

그런데 바뀌는 부분이 한 애플리케이션 안에서 동시에 여러 종류가 만들어질 수 있다면 이번엔 템플릿/콜백 패턴을 적용하는 것을 고려해볼 수 있다.

가장 전형적인 템플릿/콜백 패턴의 후보는 try/catch/finally 블록을 사용하는 코드다.

일정한 리소스를 만들거나 가져와 작업하면서 예외가 발생할 가능성이 있는 코드는 보통 try/catch/finally 구조로 코드가 만들어질 가능성이 높다.

예외상황을 처리하기 위한 catch와 리소스를 반납하거나 제거하는 finally가 필요하기 때문이다.

이런 코드가 한두 번 사용되는 것이 아니라 여기저기서 자주 반복된다면 템플릿/콜백 패턴을 적용하기 적당하다.

### 템플릿/콜백 적용
먼저 템플릿에 담을 반복되는 작업 흐름은 어떤 것인지 살펴보자.

템플릿이 콜백에게 전달해줄 내부의 정보는 무엇이고, 콜백이 템플릿에게 돌려줄 내용은 무엇인지도 생각해보자.

템플릿이 작업을 마친 뒤 클라이언트에게 전달해줘야 할 것도 있을 것이다.

템플릿/콜백을 적용할 때는 템플릿과 콜백의 경계를 정하고 템플릿이 콜백에게, 콜백에 템플릿에게 각각 전달하는 내용이 무엇인지 파악하는 게 가장 중요하다.

콜백이라는 이름이 의미하는 것처럼 다시 불려지는 기능을 만들어서 보내고 템플릿과 콜백, 클라이언트 사이에 정보를 주고받는 일이 처음에는 조금 복잡하게 느껴질지도 모르겠다.

하지만 코드의 특성이 바뀌는 경계를 잘 살피고 그것을 인터페이스를 사용해 분리한다는 가장 기본적인 객체지향 원칙에만 충실하면 어렵지 않게 템플릿/콜백 패턴을 만들어 활용할 수 있을 것이다.

## 정리
- JDBC와 같은 예외가 발생할 가능성이 있으며 공유 리소스의 반환이 필요한 코드는 반드시 try/catch/finally 블록으로 관리해야 한다.
- 일정한 작업 흐름이 반복되면서 그중 일부 기능만 바귀는 코드가 존재한다면 전략 패턴을 적용한다. </br> 바뀌지 않는 부분은 컨텍스트로, 바뀌는 부분은 전략으로 만들고 인터페이스를 통해 유연하게 전략을 변경할 수 있도록 구성한다.
- 같은 애플리케이션 안에서 여러 가지 종류의 전략을 다이나믹하게 구성하고 사용해야 한다면 컨텍스트를 이용하는 클라이언트 메소드에서 직접 전략을 정의하고 제공하게 만든다.
- 클라이언트 메소드 안에 익명 내부 클래스를 사용해서 전략 오브젝트를 구현하면 코드도 간결해지고 메소드의 정보를 직접 사용할 수 있어서 편리하다.
- 컨텍스트가 하나 이상의 클라이언트 오브젝트에서 사용된다면 클래스를 분리해서 공유하도록 만든다.
- 컨텍스트는 별도의 빈으로 등록해서 DI 받거나 클라이언트 클래스에서 직접 생성해서 사용한다. </br> 클래스 내부에서 컨텍스트를 사용할 때 컨텍스트가 의존하는 외부의 오브젝트가 있다면 코드를 이용해서 직접 DI 해줄 수 있다.
- 단일 전략 메소드를 갖는 전략 패턴이면서 익명 내부 클래스를 사용해서 매번 전략을 새로 만들어 사용하고, 컨텍스트 호출과 동시에 전략 DI를 수행하는 방식을 템플릿/콜백 패턴이라고 한다.
- 콜백의 코드에도 일정한 패턴이 반복된다면 콜백을 템플릿에 넣고 재활용하는 것이 편리하다.
- 템플릿과 콜백의 타입이 다양하게 바뀔 수 있다면 제네릭스를 이용한다.
- 스프링은 JDBC 코드 작성을 위해 JdbcTemplate을 기반으로 하는 다양한 템플릿과 콜백을 제공한다.
- 템플릿은 한 번에 하나 이상의 콜백을 사용할 수도 있고, 하나의 콜백을 여러 번 호출할 수도 있다.
- 템플릿/콜백을 설계할 때는 템플릿과 콜백 사이에 주고받는 정보에 관심을 둬야 한다.
