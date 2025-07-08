## 사용자 레벨 작업의 특징과 문제점 파악하기

사용자 데이터를 1개씩 조회하고 조건에 맞는 사용자를 1개씩 업데이트하는 방식으로 사용자 레벨 작업을 처리할 때, 중간에 예외가 발생하면 일부 데이터만 업데이트된 상태로 남는다.

1000개 데이터 중 237개가 업데이트되고 238번째에서 실패하면, 237개는 이미 변경된 상태로 남는다. 이는 은행 시스템과 같이 돈을 다루는 환경에서 대형 사고로 이어질 수 있다. 일부는 돈을 받고 일부는 못 받는 상황이 발생하기 때문이다. 이를 해결하기 위해 원자성을 가지도록 전체 작업이 성공하거나, 실패 시 모두 원래 상태로 롤백되도록 해야 한다.

## 원자성을 지키는 테스트 구현

### 테스트 시나리오 및 테스트용 대역 생성

실제 정전과 같은 상황을 재현하기 위해 네트워크 에러와 같은 예외를 시뮬레이션한다. 하지만 기존 코드를 수정해 예외를 삽입하는 것은 코드 안정성을 훼손하므로 적절하지 않다.

이를 해결하기 위해 테스트용 UserService 대역을 생성한다. TestUserService 클래스를 UserService를 상속받아 구현하며 특정 사용자 ID(targetUserId)에서 TestUserServiceException 예외를 던지도록 upgradeLevel 메서드를 오버라이드한다. 테스트 클래스 내부에 스태틱 클래스로 정의해 간편히 사용하며 private 메서드는 오버라이드 불가로 protected로 임시 변경하여 상속해서 사용할 수 있도록 한다.

```
static class TestUserService extends UserService {
    private final String targetUserId;
    public TestUserService(String targetUserId) {
        this.targetUserId = targetUserId;
    }
    @Override
    protected void upgradeLevel(User user) {
        if (user.getId().equals(targetUserId)) {
            throw new TestUserServiceException();
        }
        super.upgradeLevel(user);
    }
    static class TestUserServiceException extends RuntimeException {}
}
```

테스트 코드에서는 두 사용자(joytouch, madnite1)가 업그레이드 조건을 충족하도록 설정한다. madnite1에서 예외가 발생했을 때 joytouch도 업그레이드되지 않은 상태로 유지되어야 테스트가 성공한다. DB에서 데이터를 다시 조회해 원래 상태와 비교한다.

```
@Test
public void upgradeAllOrNothing() {
    for (User user : users) {
        userDao.add(user);
    }
    User joytouch = users.get(1);
    User madnite1 = users.get(3);
    TestUserService testUserService = new TestUserService(madnite1.getId());
    testUserService.setUserDao(userDao);
    testUserService.setUserLevelUpgradePolicy(userService.getUserLevelUpgradePolicy());
    Assertions.assertThrows(TestUserService.TestUserServiceException.class, () -> {
        testUserService.upgradeLevels();
    });
    User joytouchDb = userDao.get(joytouch.getId());
    User madnite1Db = userDao.get(madnite1.getId());
    Assertions.assertEquals(joytouch.getLevel(), joytouchDb.getLevel());
    Assertions.assertEquals(madnite1.getLevel(), madnite1Db.getLevel());
}
```

하지만 테스트는 실패한다. joytouch가 이미 업데이트된 상태로 유지되기 때문이다.

### 트랜잭션 부재에 따른 테스트 실패 원인 분석

테스트가 실패하는 이유는 upgradeLevels() 메서드의 여러 업데이트가 하나의 트랜잭션으로 묶이지 않았기 때문이다. 각 userDao.update() 호출마다 독립적인 트랜잭션이 생성되고 커밋된다.

5개 사용자를 업데이트하면 5개의 트랜잭션이 생성되며 중간에 실패해도 이전 트랜잭션은 이미 커밋된 상태다. 이를 해결하려면 모든 업데이트를 원자성을 가진 하나의 트랜잭션으로 묶어야 한다.

### 트랜잭션의 경계 설정하기

DB는 단일 SQL 명령에 대해 트랜잭션을 보장하지만 여러 SQL 명령은 하나의 트랜잭션으로 묶어야 한다.

은행 송금에서 입금과 출금이 동시에 적용되어야 하며 하나라도 실패하면 모두 롤백되어야 한다. 커밋은 모든 작업이 성공했을 때 확정하고 롤백은 예외 발생 시 모든 작업을 취소한다.

JDBC에서는 Connection 객체를 통해 트랜잭션을 관리한다. setAutoCommit(false)로 트랜잭션을 시작하고, commit() 또는 rollback()으로 종료한다.

```
Connection c = dataSource.getConnection();
c.setAutoCommit(false);
try {
    PreparedStatement st1 = c.prepareStatement("update users ...");
    st1.executeUpdate();
    PreparedStatement st2 = c.prepareStatement("delete users ...");
    st2.executeUpdate();
    c.commit();
} catch (Exception e) {
    c.rollback();
} finally {
    c.close();
}
```

트랜잭션에는 로컬 트랜잭션(단일 DB 커넥션)과 전역 트랜잭션(여러 DB 또는 리소스 포함)이 있다.

### UserService와 UserDao의 트랜잭션 문제 해결하기

현재 코드에는 트랜잭션 설정이 없으며, JdbcTemplate은 update()나 queryForObject() 호출마다 독립적인 트랜잭션을 생성/커밋한다. upgradeLevels()에서 여러 userDao.update() 호출은 각기 다른 트랜잭션으로 처리된다. 이를 해결하기 위해 모든 userDao.update()를 하나의 트랜잭션으로 묶는다.

방법 1: UserService에서 Connection을 생성하고 UserDao로 전달한다.

```
public void upgradeLevels() throws Exception {
    Connection c = dataSource.getConnection();
    c.setAutoCommit(false);
    try {
        List<User> users = userDao.getAll(c);
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(c, user);
            }
        }
        c.commit();
    } catch (Exception e) {
        c.rollback();
        throw e;
    } finally {
        c.close();
    }
}
```

이를 위해 UserDao 인터페이스를 변경한다.

```
public interface UserDao {
    void add(Connection c, User user);
    User get(Connection c, String id);
    void update(Connection c, User user);
}
```

하지만 이 방법에는 문제점이 있다:

-   JdbcTemplate을 사용할 수 없어 JDBC API를 직접 사용해야 한다.
-   Connection 파라미터로 인해 코드가 지저분해진다.
-   UserDao가 JDBC에 종속되어 JPA나 Hibernate로 변경 시 인터페이스 수정이 필요하다.
-   테스트 코드도 Connection 객체를 생성/전달해야 해 복잡해진다.

### 트랜잭션 동기화 적용하기

스프링의 트랜잭션 동기화를 통해 Connection 전달 문제를 해결한다. UserService에서 생성한 Connection을 트랜잭션 동기화 저장소에 저장하고, UserDao의 JdbcTemplate이 이를 가져와 사용한다. 트랜잭션 종료 후 동기화를 해제한다.

동작 과정:

1.  UserService가 Connection을 생성하고 트랜잭션을 시작한다(setAutoCommit(false)).
2.  Connection을 동기화 저장소에 저장한다.
3.  JdbcTemplate이 저장소에서 Connection을 가져와 SQL을 실행한다.
4.  트랜잭션 커밋/롤백 후 Connection을 해제한다.

구현:

```
public class UserService {
    private UserDao userDao;
    private DataSource dataSource;
    public void upgradeLevels() throws SQLException {
        TransactionSynchronizationManager.initSynchronization();
        Connection c = DataSourceUtils.getConnection(dataSource);
        c.setAutoCommit(false);
        try {
            List<User> users = userDao.getAll();
            for (User user : users) {
                if (canUpgradeLevel(user)) {
                    upgradeLevel(user);
                }
            }
            c.commit();
        } catch (Exception e) {
            c.rollback();
            throw e;
        } finally {
            DataSourceUtils.releaseConnection(c, dataSource);
            TransactionSynchronizationManager.unbindResource(dataSource);
            TransactionSynchronizationManager.clearSynchronization();
        }
    }
}
```

특징:

-   TransactionSynchronizationManager로 동기화 관리.
-   DataSourceUtils로 Connection 생성/해제.
-   JdbcTemplate이 동기화된 Connection을 자동 사용.

장점:

-   Connection 파라미터 제거로 코드가 깔끔해진다.
-   UserDao 인터페이스 변경 없이 기술 독립성을 유지한다.
-   멀티스레드 환경에서 스레드별 Connection 관리로 안전하다.

### 트랜잭션 서비스 추상화 도입하기

JDBC에 종속된 트랜잭션 코드는 JTA, Hibernate, JPA 등 다른 기술로 변경 시 UserService 수정이 필요하다. 이를 해결하기 위해 스프링의 PlatformTransactionManager를 사용해 트랜잭션을 추상화한다. 이를 통해 UserService는 특정 기술에 의존하지 않고 트랜잭션 경계를 설정한다.

구현:

```
public class UserService {
    private UserDao userDao;
    private PlatformTransactionManager transactionManager;
    public void upgradeLevels() {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            List<User> users = userDao.getAll();
            for (User user : users) {
                if (canUpgradeLevel(user)) {
                    upgradeLevel(user);
                }
            }
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

-   PlatformTransactionManager 인터페이스를 사용한다.
-   getTransaction()으로 트랜잭션을 시작하고, commit()/rollback()으로 종료한다.
-   DefaultTransactionDefinition으로 트랜잭션 속성(Propagation, Isolation 등)을 설정한다.

장점:

-   트랜잭션 기술 변경 시 UserService 코드 수정 없이 DI로 구현체 교체 가능.
-   DataSource 필드 제거로 완전한 기술 독립성 확보.
-   단일 책임 원칙(SRP): UserService는 비즈니스 로직만 담당.
-   개방 폐쇄 원칙(OCP): 확장에 열려 있고 수정에 닫힘.

## 메일 발송 기능 추가하기

사용자 레벨 업그레이드 시 업그레이드 안내 메일을 발송한다. 이를 위해 User 클래스와 DB users 테이블에 email 필드를 추가하고, UserDao의 insert(), update() 메서드에 email 처리를 추가한다. 기존 테스트 코드를 업데이트해 변경 사항을 검증한다.

JavaMail 사용:

```
private void sendUpgradeEmailWithSpringMailSender(User user) {
    JavaMailSenderImpl mailSender = new JavaMailSenderImpl();
    mailSender.setHost("mail.ksug.org");
    SimpleMailMessage mailMessage = new SimpleMailMessage();
    mailMessage.setFrom("master@iwaz.co.kr");
    mailMessage.setTo(user.getEmail());
    mailMessage.setSubject("Upgrade 안내");
    mailMessage.setText("사용자님의 등급이 " + user.getLevel().name() + "로 업그레이드 되었습니다.");
    mailSender.send(mailMessage);
}
```

테스트 문제:

-   SMTP 서버가 준비되어야 테스트 가능.
-   테스트마다 실제 메일 발송으로 서버 부하 증가.
-   테스트용 메일이 실제로 발송되어 비효율적.

해결책:

서비스 추상화: 스프링의 MailSender 인터페이스를 사용해 메일 발송을 추상화한다. 구현체로 JavaMailSenderImpl을 사용하고, 테스트용 대역으로 DummyMailSender와 MockMailSender를 활용한다.

구현:

```
public class UserService {
    private MailSender mailSender;
    private void sendUpgradeEmailWithSpringMailSender(User user) {
        SimpleMailMessage mailMessage = new SimpleMailMessage();
        mailMessage.setFrom("master@iwaz.co.kr");
        mailMessage.setTo(user.getEmail());
        mailMessage.setSubject("Upgrade 안내");
        mailMessage.setText("사용자님의 등급이 " + user.getLevel().name() + "로 업그레이드 되었습니다.");
        mailSender.send(mailMessage);
    }
}
```

### 테스트 대역 활용하기

테스트 대역은 테스트 환경에서 실제 의존 객체를 대체한다.

-   테스트 스텁(Test Stub): 동작을 흉내내거나 고정된 값을 반환.

```
public class DummyMailSender implements MailSender {
    @Override
    public void send(SimpleMailMessage simpleMessage) {}
    @Override
    public void send(SimpleMailMessage... simpleMessages) {}
}
```

-   목 오브젝트(Mock Object): 의존 객체와의 커뮤니케이션(입출력)을 검증.

```
public class MockMailSender implements MailSender {
    private final List<String> requests = new ArrayList<>();
    public List<String> getRequests() { return requests; }
    @Override
    public void send(SimpleMailMessage simpleMessage) {
        String[] to = simpleMessage.getTo();
        requests.addAll(Arrays.asList(Objects.requireNonNull(to)));
    }
    @Override
    public void send(SimpleMailMessage... simpleMessages) {}
}
```

테스트 코드:

```
@Test
@DisplayName("사용자 레벨 업그레이드 후 이메일 보내는지 - 목오브젝트 이용")
public void upgradeLevelsWithEmail() throws SQLException {
    for (User user : users) {
        userDao.add(user);
    }
    userService.upgradeLevels();
    checkLevelUpgraded(users.get(0), false);
    checkLevelUpgraded(users.get(1), true);
    checkLevelUpgraded(users.get(2), false);
    checkLevelUpgraded(users.get(3), true);
    checkLevelUpgraded(users.get(4), false);
    List<String> request = mockMailSender.getRequests();
    Assertions.assertEquals(request.size(), 2);
    Assertions.assertEquals(request.get(0), users.get(1).getEmail());
    Assertions.assertEquals(request.get(1), users.get(3).getEmail());
}
```

### 트랜잭션과 메일 발송 통합하기

업그레이드 중 예외 발생 시 메일 발송 여부가 불일치할 수 있다. 이를 해결하기 위해 두 가지 방법을 고려한다:

-   방법 1: 메일 발송 대상을 별도 목록에 저장하고, 업그레이드 완료 후 발송.
-   방법 2: 트랜잭션에 메일 발송을 포함해 커밋 시 발송, 롤백 시 취소.

방법 2가 로직 분리와 테스트 용이성, 확장성 면에서 우수하다.

### 설계 원칙과 DI 적용하기

-   단일 책임 원칙(SRP):
    -   UserService는 사용자 관리 로직만 담당한다.
    -   트랜잭션은 PlatformTransactionManager, 메일 발송은 MailSender가 담당한다.
-   개방 폐쇄 원칙(OCP):
    -   트랜잭션/메일 기술 변경 시 UserService 코드 수정 없이 DI로 구현체 교체 가능.
-   DI의 역할:
    -   관심사 분리: 비즈니스 로직, 데이터 액세스, 트랜잭션, 메일 발송.
    -   낮은 결합도: 기술 변경 시 영향 최소화.
    -   높은 응집도: 각 모듈이 단일 책임에 집중.

스프링 DI 활용 사례:

-   UserDao와 DataSource 분리.
-   테스트용 대역 적용.
-   트랜잭션/메일 서비스 추상화.
-   템플릿/콜백 패턴.

## 정리

-   비즈니스 로직을 담은 코드는 데이터 액세스 로직을 담은 코드와 깔끔하게 분리되는 것이 바람직함.
-   비즈니스 로직 코드 또한 내부적으로 책임과 역할에 따라 깔끔하게 메서드로 정리되어야 함.
-   DAO를 사용하는 비즈니스 로직에는 단위 작업을 보장해주는 트랜잭션이 필요함.
-   트랜잭션의 시작과 종료를 지정하는 일을 트랜잭션 경계설정이라고 함.
-   트랜잭션 경계설정은 주로 비즈니스 로직 안에서 일어나는 경우가 많음.
-   자바에서 사용되는 트랜잭션 API의 종류와 방법이 다양하므로 환경과 서버에 따라 트랜잭션 방법이 변경되면 경계설정 코드도 함께 변경돼야 함.
-   서비스 추상화는 로우레벨의 트랜잭션 기술과 API의 변화에 상관없이 일관된 API를 가진 추상화 계층을 도입.
-   테스트 대상이 사용하는 의존 오브젝트를 대체할 수 있도록 만든 오브젝트를 테스트 대역이라고 함.
-   테스트 대역 중, 테스트 대상으로부터 전달받은 정보를 검증할 수 있도록 설계된 것을 목 오브젝트라고 함.
