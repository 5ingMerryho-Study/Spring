# 목표

스프링에 적용된 템플릿 기법을 보고, 이를 이용하여 완성도 있는 DAO 코드 만드는 방법 알기

## 예외처리

코드에서 예기치 못한 오류를 잡는 것은 중요하다. 구조적으로 보았을 때 자원의 반납이 필요한 PreparedStatement 나, ResultSet, DBConnection 과 같은 작업을 할 때 예기치 못한 에러로 인해 자원이 반납되지 못한다면 서비스를 지속하기 어려워질 수 있다.

따라서 try-cath-finally 문을 이용해서 자원을 반납할 수 있다.

**수정 전**

```
public void deleteAll() throws SQLException {
	Connection c = dataSource.getConnection();
	PreparedStatement ps = c.prepareStatement("delete from users");
	ps.executeUpdate();
	//위에서 예외가 발생하면 아래는 실행되지 않는다.
	ps.close();
	c.close();
}
```

**수정 후**

```
public void deleteAll() throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;
	try {
    	c = dataSource.getConnection();
		ps = c.prepareStatement("delete from users");
		ps .executeUpdate();
	} catch (SQLException e) {
		throw e;
	} finally { 
    	if (ps != null) {
            try {
                ps.close();
            } catch (SQLException e) {
            }
        }
		if (c != null) {
            try {
                c.close();
            } catch (SQLException e) {
            }
        }
    }
}
```

복잡해졌지만 SQLException이 발생했을 때의 예외처리가 완료되었다.

**참고 : try-with-resources**

java7부터 추가된 try-with-resources를 이용한다면 자원을 반납할 때 위와 같이 복잡하게 예외처리를 하지 않아도 된다. Closeable의 부모로 AutoCloseable을 추가하여 AutoCloseable 인터페이스를 구현하고 있는 자원에 대해 try-with-resources를 적용 가능하도록 했다.

덕분에 코드가 유연해지고, 에러 처리 누락 없이 에러를 잡을 수 있게 되었다.

[https://mangkyu.tistory.com/217](https://mangkyu.tistory.com/217)

 [\[Java\] try-with-resources란? try-with-resources 사용법 예시와 try-with-resources를 사용해야 하는 이유

이번에는 오랜만에 자바 문법을 살펴보고자 합니다. Java7부터는 기존의 try-catch를 개선한 try-with-resources가 도입되었는데, 왜 try-catch가 아닌 try-with-resources를 사용해야 하는지, 어떻게 사용하는지

mangkyu.tistory.com](https://mangkyu.tistory.com/217)

## 변하는 것과 변하지 않는 것

위에서 작성한 예외처리 코드는 정성이 들어갔지만, 모든 DAO에 복잡한 예외처리문을 작성해줘야 한다는 단점이 있다. 단순히 귀찮은 것을 넘어서 복사와 붙여넣기를 하더라도 인적 오류가 발생할 가능성이 농후하다.

따라서 DB 연결과 DAO 코드를 분리하여 문제를 해결하는 방법에 대해 알아보자.

### 분리와 재사용을 위한 디자인패턴 적용

PreparedStatement를 제외한 연결과 자원 반납 부분은 변하지 않는 코드가 될 것이다. 어떤 DAO더라도 DB와 연결은 이루어질테지만, insert문인지, delete문인지에 따라 PreparedStatement문은 달라질 것이기 때문이다.

#### 템플릿 메서드 패턴 적용

상속을 통해 기능을 확장하는 템플릿 메서드 패턴을 적용해보면, 변하지 않는 부분은 슈퍼클래스에 두고 변하는 부분은 추상 메서드로 정의해서 서브클래스에 오버라이드하여 새롭게 정의해 사용할 수 있도록 할 수 있겠다.

우선, 연결 부분을 추상 메서드로 분리한다.

```
abstract protected PreparedStatement makeStatement(Connection c) throws SQLException;
```

그리고 이를 상속하는 서브클래스를 만들어 메서드를 구현하면 고정된 JDBC try-catch-finally 블록을 가진 슈퍼클래스 메서드와 필요에 따라 상속을 통해 구체적인 PreparedStatement 를 바꿔서 사용할 수 있도록 만드는 서브클래스로 깔끔하게 분리할 수 있다.

```
public class UserDaoDeleteAll extends UserDao {
	protected PreparedStatement makeStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
    }
}
```

DAO 클래스의 기능을 확장하고 싶을 때마다 상속을 통해 자유롭게 확장할 수 있고, 확장 때문에 기존의 상위 DAO 클래스에 불필요한 변화는 생기지 않도록 할 수 있다. 하지만 DAO 로직마다 상속을 통해 새로운 클래스를 만들어야한다는 단점이 있다.

또한, 확장구조가 클래스를 설계하는 시점에서 고정되어 버린다.

![image](https://github.com/user-attachments/assets/cc5e858c-f13c-4446-b030-ea5c4998bd85)


#### 전략 패턴 적용

템플릿 메서드는 상속을 통해 확장을 하지만, 전략 패턴은 오브젝트를 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만들어서 개발 폐쇄 원칙을 지키면서 유연하게 확장할 수 있다.

![image](https://github.com/user-attachments/assets/ba43b0ba-73a1-4fab-9d7f-0cb5ac3ded66)


Context의 contextMethod()에서 일정한 구조를 갖고 동작하다, 특정 확장 기능은 Strategy 인터페이스를 통해 외부의 독립된 전략 클래스에 위임한다.

위에서 살펴봤던 변하지 않는 부분이라고 명시한 것이 contextMethod()가 된다.

그 다음으로, PreparedStatement를 만들어주는 외부 기능이 전략 패턴에서의 전략이 될 수 있다. 전략 패턴의 구조에 따라 이 기능을 인터페이스로 만들고 인터페이스 메서드를 통해 PreparedStatement 생성 전략을 호출한다.

이 때 커넥션이 없으면 PreparedStatement를 만들 수 없으므로 PreparedStatement를 생성하는 전략을 호출할 때 이 컨텍스트 내에서 만들어둔 DB커넥션을 전달해야 한다.

PreparedStatement를 만드는 전략의 인터페이스는 컨텍스트가 만들어둔 Connection을 전달받아서 PreparedStatement를 만들고 만들어진 PreparedStatement 오브젝트를 돌려준다.

이 내용을 인터페이스로 정의하면 다음과 같다.

```
public interface StatementStrategy {
	PreparedStatement makePreparedStatement(Connection c) throes SQLException;
}
```

이 인터페이스를 상속해서 실제 전략, 즉 바뀌는 부분인 PreparedStatement를 생성하는 클래스를 만들어보면 다음과 같다.

```
public class DeleteAllStatement implements StatementStrategy {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
    	PreparedStatement ps = c.preparedStatement("delete from users");
        return ps;
    }
}
```

이제 이렇게 확장된 전략을 DAO 클래스에서 사용한다면 다음과 같이 사용할 수 있다.

```
public void deleteAll() throws SQLException {
	...
    try {
    	c = dataSource.getConnection();
        
        StatementStrategy strategy = new DeleteAllStatement();
        ps = stragy.makePreparedStatement(c);
        
        ps.executeUpdate();
    } catch (SQLException e) {
    ...
}
```

하지만 전략 패턴을 이렇게 이용한다면 OCP의 원칙 중 수정에는 닫혀있고 폐쇄 원칙과 확장에는 열려있다는 개방 원칙을 만족시키기 어렵다. 이렇게 컨텍스트 안에서 이미 구체적인 전략 클래스인 DeleteAllStatement를 사용하도록 고정되어 있기 때문이다.

컨텍스트가 StatementStrategy 인터페이스뿐 아니라 특정 구현 클래스인 DeleteAllStatement를 직접 알고 있다는 건, 전략 패턴에도 OCP에도 잘 들어맞는다고 볼 수 있을 것이다.

### DI 적용을 위한 클라이언트/컨텍스트 분리

전략 패턴에 따르면 Context가 어떤 전략을 사용하게 할 것인가는 Context를 사용하는 앞단의 Client가 결정하는 게 일반적이다. Client가 구체적인 전략의 하나를 선택하고 오브젝트로 만들어서 Context에 전달하는 것이다. Context는 전달받은 그 Strategy 구현 클래스의 오브젝트를 사용한다.

![image](https://github.com/user-attachments/assets/3635066a-2943-45e3-a414-02f14a7cf6a9)


컨텍스트에 해당하는 부분을 별도의 메서드로 독립시킨 코드 예제이다.

![image](https://github.com/user-attachments/assets/2c0d3a48-d918-41e3-b6d1-72e8e100625d)


## JDBC 전략 패턴의 최적화

### 익명 내부 클래스

로컬 클래스로 사용하던 내부 클래스인 AddStatement 클래스는 add()라는 메서드에서만 사용할 용도로 만들어졌다고 했을 때, 간결하게 클래스 이름을 제거한 익명 내부 클래스로 만들 수 있다.

> **익명 내부 클래스  
> **익명 내부 클래스는 이름을 갖지 않는 클래스이다. 클래스 선언과 오브젝트 생성이 결합된 형태로 만들어지며, 상속할 클래스나 구현할 인터페이스를 생성자 대신 사용해서 다음과 같은 형태로 만들어 사용한다. 클래스를 재사용할 필요가 없고, 구현한 인터페이스 타입으로만 사용할 경우에 유용하다.  
>   
> new 인터페이스이름() { 클래스 본문 };

![image](https://github.com/user-attachments/assets/6dddc409-0b16-440c-9310-80bbba69d726)


## 컨텍스트와 DI

### JdbcContext의 분리

전략 패턴의 구조로 보면 UserDao의 메서드가 클라이언트이고, 익명 내부 클래스로 만들어지는 것이 개별적인 전략, jdbcContextWithStatementStrategy() 메서드는 컨텍스트다. 컨텍스트 메서드는 UserSao 내의 PreparedStatement를 실행하는 기능을 가진 메서드에서 공유할 수 있다. 하지만 jdbcContextWithStatementStrategy()는 JDBC의 일반적인 작업 흐름을 담고 있기 때문에 다른 DAO에서도 사용할 수 있을 것이다.

따라서 독립시켜 사용해보자.

#### 클래스 분리

```
public class JdbcContext {
	private DataSource dataSource;
    
    public void setDataSource(DataSource dataSource) {
    	this.dataSource = dataSource;
    }
    
    public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
    	Connection c = null;
        PreparedStatement ps = null;
        
        try {
        	c = this.dataSource.getConnection();
            
            ps = stmt.makePreparedStatement(c);
            
            ps.executeUpdate();
        } catch (SQLException e) {
        	throw e;
        } finally {
        	if(ps != null) { try { ps.close(); } catch (SQLException e) {} }
            if(c != null) { try { c.close(); } catch (SQLException e) {} }
        }
    }
}
```

위의 코드를 분리된 JdbcContext를 DI 받아서 사용할 수 있게 만든다.

![image](https://github.com/user-attachments/assets/6fba6926-42e7-43a1-9d64-b61bffeb728e)


관계를 그려보면 다음과 같다.

![image](https://github.com/user-attachments/assets/82710d05-4f03-4200-8073-dd6877b98b27)


## 템플릿과 콜백

### 템플릿/콜백의 동작원리

템플릿은 고정된 작업 흐름을 가진 코드를 재사용하는 것이고 콜백은 템플릿 안에서 호출되는 것을 목적으로 만들어진 오브젝트를 말한다.

#### 템플릿/콜백의 특징

여러 개의 메서드를 가진 일반적인 인터페이스를 사용할 수 있는 전략 패턴의 전략과 달리 템플릿/콜백 패턴의 콜백은 보통 단일 메서드 인터페이스를 사용한다.

콜백 인터페이스의 메서드에는 보통 파라미터가 있고 템플릿 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달받을 때 사용된다.

![image](https://github.com/user-attachments/assets/2f55dc1c-0fd3-4559-b224-31bd453c6de0)


DI 방식의 전략 패턴 구조라고 볼 수 있다.

## 정리

-   JDBC와 같이 예외가 발생할 가능성이 있으며 공유 리소스의 반환이 필요한 코드는 반드시 try-catch-finally 블록으로 관리해야 한다.
-   일정한 작업 흐름이 반복되며 그중 일부 기능만 바뀌는 코드가 존재한다면 전략 패턴을 적용한다. 바뀌지 않는 부분은 컨텍스트로, 바뀌는 부분은 전략으로 만들고 인터페이스를 통해 유연하게 전략을 변경할 수 있도록 구성한다.
-   같은 어플리케이션 안에서 여러 가지 종류의 전략을 동적으로 구성하고 사용해야 한다면 컨텍스트를 이용하는 클라이언트 메서드에서 직접 전략을 정의하고 제공하게 만든다.
-   클라이언트 메서드 안에 익명 내부 클래스를 사용해서 전략 오브젝트를 구현하면 코드도 간결해지고 메서드의 정보를 직접 사용할 수 있어서 편리하다.
-   컨텍스트가 하나 이상의 클라이언트 오브젝트에서 사용된다면 클래스를 분리해서 공유하도록 만든다.
-   컨텍스트는 별도의 빈으로 등록해서 DI 받거나 클라이언트 클래스에서 직접 생성해서 사용한다. 클래스 내부에서 컨텍스트를 사용할 때 컨텍스트가 의존하는 외부의 오브젝트가 있다면 코드를 이용해서 직접 DI 해줄 수 있다.
-   단일 전략 메서드를 갖는 전략 패턴이면서 익명 내부  클래스를 사용해서 매번 전략을 새로 만들어 사용하고, 컨텍스트 호출과 동시에 전략 DI를 수행하는 방식을 템플릿/콜백 패턴이라고 한다.
-   콜백의 코드에도 일정한 패턴이 반복된다면 콜백을 템플릿에 넣고 재활용하는 것이 편리하다.
-   템플릿과 콜백의 타입이 다양하게 바뀔 수 있다면 제네릭스를 이용한다.
-   스프링은 JDBC 코드 작성을 위해 JdbcTemplate을 기반으로 하는 다양한 템플릿과 콜백을 제공한다.
-   템플릿은 한 번에 하나 이상의 콜백을 사용할 수도 있고, 하나의 콜백을 여러 번 호출할 수도 있다.
-   템플릿/콜백을 설계할 때는 템플릿과 콜백 사이에 주고받는 정보에 관심을 둬야 한다.
