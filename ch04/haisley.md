# Chapter 4

## 하지 말아야 할 것

1. 예외를 잡고 아무 처리도 하지 않는 것 
    
    → 발생한 예외로 인해 어떤 기능이 비정상적으로 동작
    
    → 메모리나 자원 소진
    
    → 예상치 못한 다른 문제 발생
    
2. 예외를 잡고 화면에 예외를 출력 (`System.out.println()` , `e.printStackTrace()`)
    
    → 운영 서버에서 콘솔 로그를 모니터링하지 않는 한 확인이 어려움
    
    → 화면에 단순히 메시지를 출력한 것은 실제로 예외를 처리한 게 아님 
    
3. 무책임한 `throws` 선언
    
    → `throws Exception` 코드는 의미 있는 정보를 줄 수 없어, 해당 메서드를 사용하는 다른 메서드에도 같은 코드를 넣을 수밖에 없음
    
    → 적절한 처리를 통해 복구될 수 있는 예외상황도 제대로 다룰 수 없게 됨
    

## 예외의 종류와 특징

1. Error
    - `java.lang.Error` 클래스의 서브클래스들
    - 주로 자바 VM에서 발생시키는 것으로, 애플리케이션 코드에서 잡으려고 하면 안됨
2. Exception
    - `java.lang.Exception` 클래스와 그 서브클래스로 정의되는 예외들
    - 애플리케이션 코드의 작업 중에 예외상황이 발생했을 경우에 사용됨
    - CheckedException VS UncheckedException
        - **CheckedException**
            - 처리되지 않으면 컴파일 에러 발생
            - ex ) `IOException` , `SQLException`
        - **UncheckedException**
            - `java.lang.RuntimeException` 클래스를 상속한 런타임 예외들은 명시적인 예외처리를 강제하지 않음
            - 주로 프로그램의 오류가 있을 때 발생하도록 의도된 것들
            - ex ) `NullPointerException` , `IllegalArgumentException`

## 예외 처리 방법

- 예외 복구
    - 재시도가 의미있는 경우라면 최대 횟수만큼 반복적으로 시도
- 예외 처리 회피
    - 예외를 처리하지 않고, 자신을 호출한 쪽으로 던져버리는 것
- 예외 전환
    1. 중첩 예외로 만들기
        - 보통 전환하는 예외에 원래 발생한 예외를 담아서 만듦
    2. 예외를 포장하기
        - 주로 예외처리를 강제하는 체크 예외를 언체크 예외인 런타임 예외로 바꾸는 경우
    
    복구 가능 예외는 `try/catch` 를 사용해 에러를 처리해야 하고, 복구가 불가능한 예외는 런타임 예외로 포장해서 메서드 밖으로 `throw` 해서 에러를 처리한다.
    
    `런타임 예외 처리` 의 경우 컴파일 시점에 예외를 강제하지 않기 때문에 `개발자가 예외 상황을 고려하지 못할 가능성` 이 있다.
    

## 예외처리 전략

### 런타임 예외의 보편화

- 자바 엔터프라이즈 서버환경에서는 수많은 사용자가 동시에 요청을 보내고 각 요청이 독립적인 작업으로 취급됨
- 하나의 요청을 처리하는 중에 예외가 발생하면, 애플리케이션이 종료되지 않도록 상황을 복구할 필요가 없어짐
- 애플리케이션 차원에서 예외상황을 미리 파악하고, 예외가 발생하지 않도록 차단하는 게 좋음
- 대응이 불가능한 체크 예외라면 빨리 런타임 예외로 전환해서 던지는 게 좋음

### 애플리케이션 예외

- 시스템 또는 외부의 예외상황이 원인이 아니라 애플리케이션 자체의 로직에 의해 의도적으로 발생시키고, 반드시 catch 해서 무엇인가 조치를 취하도록 요구하는 예외
- 예외 처리 방법
    1. 메서드의 리턴 값을 기반으로 IF 분기로 예외 상황 체크하기
    2. 정상적인 흐름을  따르는 코드는 그대로 두고, 예외 상황에서는 의미 있는 예외를 던지기

## 예외 전환

### 예외 전환의 목적

- 런타임 예외로 포장해서 던지면 호출하는 쪽에서 필요하지 않은 `try/catch` 를 줄여주는 것
- 로우레벨의 예외를 좀 더 의미 있고 추상화된 예외로 바꿔서 던져주는 것

### JDBC API의 한계

- 비표준 SQL : DB에 특화된 기능이나 최적화를 위한 DB 종속적 SQL 쿼리
- 호환성 없는 SQLException의 DB 에러 정보 : DB에 따라 발생할 수 있는 예외의 원인과 종류가 다름

### DB에 독립적으로 예외 처리

- 스프링은 `DataAccessException`과 서브 클래스로 세분화된 예외 제공
    - DB별 에러 코드를 분류해서 스프링이 정의한 예외 클래스와 매핑해놓은 에러 코드 매핑정보 테이블을 사용 → `JdbcTemplate` 은 DB 종류에 상관없이 동일한 예외를 받을 수 있음
        
        ```xml
        <bean id="Oracle" class="org.springframework.jdbc.support.SQLErrorCodes">
            <property name="badSqlGrammerCodes">
                <value> 900, 903, 904, 917, 936, 942, 17006</value>
            </property>
            <property name="invalidResultSetAccessCodes">
                <value> 17003 </value>
            </property>
            <property name="duplicateKeyCodes">
                <value> 1 </value>
            </property>
            <property name="dataIntegrityViolationCodes">
                <value> 1400, 1722, 2291, 2292 </value>
            </property>
            <property name="dataAccessResourceFailureCodes">
                <value> 17002, 17447 </value>
            </property>
            ...
        </bean>
        ```
        

### 데이터 액세스 기술에 따른 예외 처리

- 스프링 `DataAccessException` 은 JDBC 이외에 자바 데이터 액세스 기술에서 발생하는 예외에도 적용
    - `JPA` (ORM)
    - `MyBatis` (SQL Mapper)

## 정리

- 예외를 잡아서 아무런 조치를 취하지 않거나 의미 없는 `throws` 선언을 남발하는 것은 위험하다.
- 예외는 복구하거나 예외처리 오브젝트로 의도적으로 전달하거나 적절한 예외로 전환해야 한다.
- 좀 더 의미 있는 예외로 변경하거나, 불필요한 `catch/throws`를 피하기 위해 런타임 예외로 포장하는 두 가지 방법의 예외 전환이 있다.
- 복구할 수 없는 예외는 가능한 한 빨리 런타임 예외로 전환하는 것이 바람직하다.
- 애플리케이션의 로직을 담기 위한 예외는 체크 예외로 만든다.
- JDBC의 `SQLException`의 에러 코드는 DB에 종속되기 때문에 DB에 독립적인 예외로 전환될 필요가 있다.
- 스프링은 `DataAccessException`을 통해 DB에 독립적으로 적용 가능한 추상화된 런타임 예외 계층을 제공한다.
- DAO를 데이터 액세스 기술에서 독립시키려면 인터페이스 도입과 런타임 예외 전환, 기술에 독립적인 추상화된 예외로 전환이 필요하다.
