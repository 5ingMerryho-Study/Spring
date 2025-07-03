객체지향 설계의 핵심 원칙인 **개방 폐쇄 원칙**은, 변화의 특성이 다른 부분을 구분해주고, 각각 다른 목적과 다른 이유에 의해 다른 시점에 독립적으로 변경될 수 있는 효율적인 구조를 만들어주는 것이다.

**템플릿**이란, 바뀌는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법이다.

### 3.1 다시 보는 초난감 DAO

- JDBC 코드에서 예외가 발생했을 경우에도 사용한 리소스를 반드시 반환하도록 예외처리가 필요하다.
- 리소스 반환과 `close()`
    - Connection이나 PreparedStatement에는 사용한 리소스를 풀로 다시 돌려주는 역할을 하는 close() 메서드가 있다.
    - Connection과 PreparedStatement는 보통 미리 정해진 풀 안에 제한된 수의 리소스를 만들어 두고 필요할 때 이를 할당하고, 반환하면 다시 풀에 넣는 방식으로 운영된다.
    - 요청이 매우 많은 서버환경에서는 매번 새로운 리소스를 생성하는 대신 풀에 미리 만들어둔 리소스를 돌려가며 사용하는 편이 훨씬 유리하다.

- try-catch-finally 구문을 적용한 JDBC 예외처리 (delete)
    
    ```java
    public void deleteAll() throws SQLException {
    	Connection c = null;
    	PreparedStatement ps = null;
    	
    	try {
    		c = dataSource.getConnection();
    		ps = c.prepareStatement("delete from users");
    		ps.executeUpdate();
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
    
- try-catch-finally 구문을 적용한 JDBC 예외처리 (select)
    
    ```java
    public int getCount() throws SQLException {
    	Connection c = null;
    	PreparedStatement ps = null;
    	ResultSet rs = null;
    	
    	try {
    		c = dataSource.getConnection();
    		ps = c.prepareStatement("select count(*) from users");
    		
    		rs = ps.executeQuery();
    		rs.next();
    		return rs.getInt(1);
    	} catch (SQLException e) {
    		throw e;
    	} finally {
    		if (rs != null) {
    			try {
    				rs.close();
    			} catch (SQLException e) {
    			}
    		}
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
    

### 3.2 변하는 것과 변하지 않는 것

- JDBC try/catch/finally 코드의 문제점
    - 중첩된 try/catch/finally 코드로 복잡함
- 중복되는 코드와 로직에 따라 확장되고 자주 변하는 코드를 분리해야 함
    - 분리와 재사용을 위한 개선 방법
        - 메서드 추출
            
            ```java
            // 변하는 부분을 메서드로 추출한 후의 deleteAll()
            public void deleteAll() throws SQLException {
            	...
            	try {
            		c = dataSource.getConnection();
            		
            		ps = makeStatement(c);
            		
            		ps.executeUpdate();
            	} catch (SQLException e) {
            		...
            	}
            }
            
            private PreparedStatement makeStatement(Connection c) throws SQLException {
            	PreparedStatement ps;
            	ps = c.prepareStatement("delete from users");
            	return ps;
            }
            ```
            
            - **분리시킨 메서드를 다른 곳에서 재사용할 수 있어야** 하나, 분리된 메서드가 DAO 로직마다 새롭게 확장되어야 하는 문제가 있음
        - 템플릿 메서드 패턴의 적용
            
            ```java
            abstract protected PreparedStatement makeStatement(Connection c) throws SQLException;
            
            public class UserDaoDeleteAll extends UserDao {
            	protected PreparedStatement makeStatement(Connection c) throws SQLException {
            		PreparedStatement ps = c.prepareStatement("delete from users");
            		return ps;
            	}
            }
            ```
            
            - 상속을 통해 UserDao 클래스 기능 자유롭게 확장 가능
            - Dao 로직마다 상속을 통해 새로운 클래스를 만들어야 함
            - 확장구조가 이미 클래스를 설계하는 시점, 즉 컴파일 시점에 고정되어 유연성이 떨어짐
        - 전략 패턴의 적용
            - 오브젝트를 2개로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존
            - OCP 개방 폐쇄 원칙 준수, 템플릿 메서드 패턴보다 유연하고 뛰어난 확장성
            - 확장 부분을 별도의 클래스로 만들어 추상화된 인터페이스를 통해 위임하는 방식
            
            ```java
            public interface StatementStrategy {
            	PreparedStatement makePreparedStatement(Connection c) throws SQLException;
            }
            
            public class DeleteAllStatement implements StatementStrategy {
            	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
            		PreparedStatement ps = c.prepareStatement("delete from users");
            		return ps;
            	}
            }
            ```
            
            ```java
            public void deleteAll() throws SQLException }
            	...
            	try {
            		c = dataSource.getConnection();
            		
            		StatementtStrategy strategy = new DeleteAllStatement();
            		ps = strategy.makePreparedStatement(c);
            		
            		ps.executeUpdate();
            	} catch (SQLException e) {
            		...
            	}
            }
            ```
            
        - DI 적용을 위한 클라이언트/컨텍스트 분리
            - context가 어떤 전략을 사용하게 할 것인가는 client가 결정하는 게 일반적
            - context가 사용할 구체적인 전략을 client가 선택하고 오브젝트로 만들어서 전달
            - context는 전달받은 strategy 구현 클래스의 오브젝트를 사용
            - 전략 패턴의 장점을 일반적으로 활용할 수 있도록 만든 구조가 DI
            
            ```java
            // 메서드로 분리한 try/catch/finally 컨텍스트 코드
            public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
            	Connection c = null;
            	PreparedStatement ps = null;
            	
            	try {
            		c = dataSource.getConnection();
            		ps = stmt.makePreparedStatement(c);
            		
            		ps.executeUpdate();
            	} catch (SQLException e) {
            		throw e;
            	} finally {
            		if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
            		if (c != null) { try { c.close(); ) catch (SQLException e) {} }
            	}
            }
            ```
            
            ```java
            public void deleteAll() throws SQLException {
            	StatementStrategy st = new DeleteAllStatement(); // 선정한 전략 클래스의 오브젝트 생성
            	jdbcContextWithStatementStrategy(st); // 컨텍스트 호출. 전략 오브젝트 전달.
            }
            ```
            
        - 마이크로 DI
            - 매우 작은 단위의 코드와 메서드 사이에서 일어나는 DI
            - 제3자의 도움으로 두 오브젝트 사이의 유연한 관계를 설정하는 DI의 형태 중 하나

### 3.3 JDBC 전략 패턴의 최적화

- 중첩 클래스의 종류
    - static : 독립적인 오브젝트로 만들어질 수 있는 클래스
    - inner : 자신이 정의된 클래스의 오브젝트 안에서만 만들어질 수 있는 클래스
        - member inner : 멤버 필드처럼 오브젝트 레벨에 정의되는 클래스
        - local : 메서드 레벨에 정의되는 클래스
            - 클래스 파일이 많아지는 문제 →  UserDao 클래스의 내부 클래스로 정의해 해결
            - 로컬 클래스가 내부 클래스이기 때문에 자신이 선언된 것의 정보에 접근할 수 있음
            - 내부 클래스에서 외부의 변수를 사용할 때는 외부 변수를 반드시 final로 선언
        - anonymous inner : 이름을 갖지 않는 클래스, 선언된 위치에 따라 범위가 다름
            - 클래스 이름을 제거하여 더 간결하게 만들 수 있음
            - 클래스 선언과 오브젝트 생성이 결합된 형태로 만들어짐
            - 상속한 클래스나 구현할 인터페이스 생성자 대신 사용
            - 클래스를 재사용할 필요 X, 구현한 인터페이스 타입으로만 사용할 경우 유용

### 3.4 컨텍스트와 DI

- 스프링 빈으로 DI
    - 객체의 생성과 관계 설정에 대한 제어 권한을 오브젝트에서 제거하고 외부로 위임 (IoC)
    - 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈으로 설정
    - 오브젝트 간 실제 의존관계가 설정파일에 명확하게 드러남
    - 구체적인 클래스와의 관계가 설정에 직접 노출됨
- 코드를 이용하는 수동 DI
    - UserDao 내부에서 직접 DI를 적용하는 방법
    - UserDao가 JdbcContext에 대한 제어권, 생성, 관리, DI를 맡김
    - 한 오브젝트의 수정자 메서드에서 다른 오브젝트를 초기화하고 코드를 이용해 DI
    - 의존관계가 외부에 드러나지 않음

### 3.5 템플릿과 콜백

- 바뀌지 않는 일정한 패턴을 갖는 작업 흐름이 존재, 일부만 자주 바꾸어 사용해야 하는 경우 적합한 구조
- 템플릿 (고정된 틀 안에 바꿀 수 있는 부분을 넣어서 사용)
- 콜백 (실행되는 것을 목적으로 다른 오브젝트의 메서드에 전달되는 오브젝트)
- 템플릿/콜백의 동작원리
    - 보통 단일 메서드 인터페이스 사용
    - 콜백은 일반적으로 하나의 메서드를 가진 인터페이스를 구현한 익명 내부 클래스로 만들어짐
    - 콜백 인터페이스의 메서드에는 보통 템플릿의 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달 받을 때 사용)
    - 매번 메서드 단위로 사용할 오브젝트를 새롭게 전달 받음
    - 콜백 오브젝트가 내부 클래스로서 자신을 생성한 클라이언트 메서드 내의 정보를 직접 참조
- 콜백의 분리와 재활용
    - 분리를 통해 재사용 가능한 코드를 찾아낼 수 있다면 익명 내부 클래스를 사용한 코드 간결하게 변환
- 콜백과 템플릿의 결합
    - 재사용 가능한 콜백을 담고 있는 메서드라면 Dao가 공유할 수 있는 템플릿 클래스 내부로 옮기기
    - 구체적인 구현과 내부의 전략 패턴, 코드에 의한 DI, 익명 내부 클래스 등의 기술은 감추기
    - 외부에는 꼭 필요한 기능을 제공하는 단순 메서드만 노출

### 3.6 스프링의 JdbcTemplate

- 네거티브 테스트부터 만드는 것이 좋음 (예외상황에 대한 테스트)
- 재사용 가능한 콜백의 분리
    - 예외처리, 리소스 반납, DB 연결 방식 등에 대한 책임과 관심은 모두 JdbcTemplate에 있음
    - 책임이 다른 코드와 낮은 결합도를 유지
    - 특정 템플릿/콜백 구현에 대한 강한 결합
