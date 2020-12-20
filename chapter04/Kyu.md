# Chapter 04 예외

탬플릿이란? 바뀌는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로 독립시켜서 효과적으로 활용할 수 있도록 하는 방법

4-1 사라진 SQLExcetpion
----------------------------------------------------------------------------
3장에서 JdbcContext로 만들었던 코드를 스프링의 JdbcTemplate을 적용하도록 바꾸면서 throw SQLException 이 사라졌다.

```java
public void deleteAll() throws SQLException {
    this.jdbcContext.executeSql("delete from users");
}

=> jdbcTemplate 적용

public void deleteAll() {
    this.jdbcTemplate.update("delete from users");
}
```

예외 블랙홀

```java
try {
...
} catch(SQLException e) {}
```
예외가 발생하면 catch 블록을 써서 잡아내는 것까지는 좋은데 아무것도 하지 않고 별문제 없는 것처럼 넘어가 버리는 건 위험한 일이다.
프로그램 실해중에 어디선가 오류가 있어서 예외가 발생하는데 그것을 무시하고 게속 진행해버리기 떄문에 원치 않는 예외가 발생하는 것보다도 훨씬 나쁜 일이다.

```java
try {
...
} catch(SQLException e) {System.out.println(e);}

try {
...
} catch(SQLException e) {e.printStackTrace();}
```
예외가 발생하면 출력해주는 코드인데, 출력해봐야 다른 로그나 메세지에 금방 묻혀버릴 수 있다. 
=> 모든 예외는 적절하게 복구되던지 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보되어야 한다.


무의미하고 무책임한 throw 

```java
public void method1() throws Exception {
  method2();
}
public void method2() throws Exception {
  method3();
}
public void method3() throws Exception {
  ...
}
```
어떤 예외인지 분류하지도 않고 무조건 throw Exception 라는 모든 예외를 무조건 던져버리는 선언을 메소드에 기계적으로 넣은 것 뿐이다.
이런 메소드 선언에서는 의미 있는 정보를 얻을 수 없다.


예외의 중류와 특징

자바에서 throw를 통해 발생시킬 수 있는 예외는 크게 3가지가 있다.

- Error

 첫째는 java.lang.Error 클래스의 서브클래스들이다. 에러는 시스템에 뭔가 비정상적인 상황이 발생했을 경우에 사용된다. 주로 JVM에서 발생시키는 것이기 때문에 어플리케이션 코드에서 catch 블록으로 잡아봐야
 대응할 수 있는 방안이 딱히 없다. 따라서 시스템 레벨에서 특별한 작업을 하는게 아니라면 Error에 대한 처리는 신경쓰지 않아도 된다.
 
 - Checked Exception
 
 둘째는 java.lang.Exception 클래스의 서브클래스들 중 RuntimeException을 상속하지 않는 예외이다. 체크예외가 발생할 수있는 메소드들을 사용할 경우 반드시 예외를 처리하는 코드를 함께 작성해야만 한다.
 
 - Unchecked Exception(RuntimeException)
 
 셋째는 java.lang.Exception 클래스의 서브클래스들 중 RuntimeException을 상속하는 예외이다. 명시적인 예외처리를 강제하지 않는다. 주로 어플리케이션의 오류가 있을 때 발생한다. 피할 수 있지만 개발자가
 부주의해서 발생할 수 있는 경우에 발생하도록 만든 것이 런타임 예외이다. 따라서 런타임 예외는 예상하지 못했던 예외상황에서 발생하는 게 아니기 때문에 굳이 catch나 throws를 하지 않아도 된다.
 
 
예외처리 방법

- 예외 복구

예외상황을 파악하고 문제를 해결해서 정상 상태로 돌려놓는 것이다. (재시도, 재입력 등등..) 예외처리 코드를 강제하는 체크 예외들은 이렇게 예외를 어떤식으로도 복구할 가능성이 있는 경우에 사용한다. 

- 예외 회피

예외처리를 자신이 담당하지 않고 호출한 쪽으로 다시 던져버리는 것이다. 다른 오브젝트나 메소드가 예외를 대신 처리할 수 있도록 아래와 같이 그대로 던져줘야 한다.
예외를 회피하는 것은 예외를 복구하는 것처럼 의도가 분명해야 한다. 콜백/템플릿 처럼 긴밀한 관계에 있는 다른 오브젝트에게 예외처리 책임을 분명히 지게 하거나, 자신을 사용하는 쪽에서 예외를 다루는게 최선이라는
확신이 있어야 한다.

```java
public void add() throws SQLException {
  // JDBC API
}

public void add() throws SQLException {
  try {
    // JDBC API
  } catch(SQLException e) {
    // 로그 출력
    throw e;
  }
}
```

- 예외 전환

예외 회피와 비슷하게 예외를 복구해서 정상적인 상태로 만들수 없기 때문에 예외를 밖으로 던지는것이다. 하지만 발생한 예외를 그대로 넘기는 게 아니라 적절한 예외로 전환해서 던진다.
예외전환은 보통 두가지 목적으로 사용된다.
1. 내부에서 발생한 예외를 그대로 던지는 것이 그 예외 상황에 대한 적절한 의미를 부여해주지 못할 때 의미를 분명하게 해줄 수 있는 예외로 바꿔주기 위함이다.
보통 전환하는 예외에 원래 발생한 예외를 담아서 nested exception으로 만드는 것이 좋다.
```java
catch(SQLException e) {
  ...
  throw DuplicateUserIdException(e);
}

catch(SQLException e) {
  ...
  throw DuplicateUserIdException().initCause(e);
}
```
2. 예외를 처리하기 쉽고 단순하게 만들기 위해 포장하는 것이다. 원인이 되는 예외를 내부에 담아서 nested exception 으로 던지는 것은 같지만 의미 명확하게 하기 위함이 아닌
 의미도 없고 복구도 불가능한 체크 예외를 언체크 예외로 바꾸기 위해 사용된다.
```java
try {
  OrderHome orderHome = EJBHomeFactory.getInstance().getOrderHome();
  Order order = orderHome.findByPrimaryKey(Integer id);
} catch (NamingException ne) { // 체크 예외
  throw new EJBException)(ne);
} catch (SQLException se) { // 체크 예외
  throw new EJBException(se);
} catch (RemoteException re) { // 체크 예외
  throw new EJBException(re);
}
```
일반적으로 체크 예외를 계속 throw를 사용해 넘기는건 무의미하다. 어짜피 복구가 불가능한 예외라면 가능한 빨리 런타임 예외로 포장해 다른 계층의 메소드를 작성할 때 불필요한 throws 선언을 방지해 준다.
어짜피 복구하지 못할 예외라면 어플리케이션 코드에서는 런타임 에러로 포장해서 던지고 예외처리 서비스 등을 이용해 자세한 로그를 남기고, 관리자에게 메일 등을 통해 통보해주고, 사용자에게는 안내 메세지를 남기는 등의 처리를 해주는 것이 적절하다.


예외처리 전략

런타임 예외의 보편화

체크예외는 복구할 가능성이 조금이라도 있는 예외적인 상황이기 떄문에 자바는 이를 처리하는 catch블록이나 throws선언을 강제한다. 문제는 독립형 어플리케이션이 아닌 서버의 특정 계층에서 예외가 발생했을때 작업을
중지하고 사용자와 바로 커뮤니케이션 하면서 예외 상황을 복구할 수는 없다. 차라리 어플리케이션 차원에서 예외상황을 미리 파악하여 예외가 발생하지 않도록 차단하는것이 좋다. 
자바의 환경이 서버로 이동하면서 체크 예외의 활동도와 가치는 점점 떨어지고 있다. 그래서 대응이 불가능한 체크 예외라면 런타임 예외로 전환해서 던지는게 낫다.
예전에는 복구할 가능성이 조금이라도 있다면 체크 예외로 만드는 경향이 있었는데, 지금은 항상 복구할 수 있는 예외가 아니라면 일단 언체크 예외로 만들고 필요하다면 언제든 catch 블록으로 잡아서 처리하는 경향이 있다.


add 메소드의 예외처리

add 메소드는 DuplicatedUserIdException, SQLException 두가지 예외를 던지게 되어 있다.
JDBC 코드에서 SQLException이 발생할 수 있는데 그 원인이 ID 중복이면 더 의미있는 DuplicatedUserIdException 으로 전환해주고 아니면 그대로 SQLException 을 던지게 했다.

DuplicatedUserIdException는 굳이 체크 예외로 둬야하는 것은 아니다. DuplicatedUserIdException 처럼 의미 있는 예외는 add 메소드를 호출한 오브젝트 대신 더 앞단의 오브젝트에서 다룰 수 있다.
어디에서는 DuplicatedUserIdException를 처리할 수 있다면 굳이 체크 예외로 두지 않고 런타임 에외로 만드는게 낫다
또한, SQLException은 대부분 복구 북가능한 예외이므로 잡아봐야 처리할 것도 없고 결국 throw를 타고 앞으로 전달만 되다가 어플케이션 밖으로 던져질 것이다. 그럴바에 차라리 런타임 예외로 포장해서 밖의 메소드들이
의미없는 throw를 하지 않도록 막아주는게 좋다.

필요하다면 언제든 잡아서 처리할 수 있도록 별도의 예외로 정의하기는 하지만 필요 없다면 신경쓰지 않아도 되도록 RuntimeException을 상속한 런타임 예외로 만든다. 중첩 예외를 만들 수 있도록 생성자를 추가해주는것을 잊지 말자

```java
public class DuplicatedUserIdException extends RuntimeException {
  public DuplicatedUserIdException(Throwable cause) {
    super(cause);
  }
}

public void add(final User user) throws DuplicatedUserIdException {
    try {
      ...
    } catch (SQLException e) {
      if(e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
        throw new DuplicatedUserIdException(e); // 예외 전환
      else
        throw new RuntimeException(e); // 예외 포장
    }
}
```

이제 add 메소드를 사용하는 오브젝트들은 SQLException을 처리하기 위해 불필요한 throws 선언을 할 필요는 없으면서 필요하면 아이디 중복을 처리하기 위해 catch 블록을 통해 DuplicatedUserIdException을 사용할 수 있다.
단, 런타임 예외로 만들었기 때문에 사용에 더 주의를 기울여야 한다. 컴파일러가 예외처리를 강제하지 않기 때문에 문서나 레퍼런스를 통해 메소드를 사용할 때 발생할 수 있는 예외의 종류와 원인에 대하 설명해두자.


어플리케이션 예외

시스템 또는 외부의 예외상황이 원인이 아니라 어플리케이션 자체의 로직에 의해 의도적으로 발생시키고 반드시 catch 해서 무엇인가 조치를 취하도록 요구하는 예외를 어플리케이션 예외라고 한다.
어플리케이션 예외를 처리하는 방법은
1. 정상처리 된 경우와 예외경우에 다른 종류의 리턴값을 돌려준다.
물론 이것은 시스템 오류가 아니기 때문에 두가지 모두 정상적인 흐름이다. 예외상황에 대한 리턴 값을 명확하게 코드화하고 잘 관리하지 않으면 혼란이 생길 수있다. 또한 리턴 값을 확인하는 조건문이 자주 등장한다는 문제도 있다.
2. 정상적인 흐름을 타는 코드는 그대로 두고 예외 상황에서 비지니스적인 의미를 띈 예외를 던지도록 만든다. (이때 예외는 체크 예외로 만든다)
예외 상황에 대한 로직 구현을 강제하도록 만드는 것이다.
```java
try {
  BigDecimal balance = account.withdraw(amount);
  ...
  // 정상적인 처리 결과를 출력하도록 진행
} catch(InsufficientBalanceException e) { // 체크 예외
  BigDecimal avilFunds = e.getAvailFunds();  // 인출 가능 잔고를 InsufficientBalanceException 에서 가져옴
  ...
  // 후 처리
}
```

SQLException은 어떻게 됐나?

지금까지 다룬 예외처리에 대한 내용은 JdbcTemplate을 적용하는 과정에 throws SQLExcetpion이 왜 사라졌는가에 대한 이유를 설명하기 위한 것이었다. 스프링의 예외처리 전략과 원칙을 알고 있어야 하기 때문이다.
다시 SQLException에 대해서 생각해보면 대부분의 SQLException은 복구가 불가능하다. 따라서 예외처리 전략을 적용해야 한다.
필요도 엇는 throws를 반복하지 말고 가능한 빨리 런타임 예외로 전환해줘야한다.
스프링의 JdbcTemplate은 바로 이 전략을 따르고 있다. JdbcTemplate과 콜백 안에서 발생하는 모든 SQLException을 런타임 예외인 DataAccessException으로 포장해서 던져주고 있다.
따라서 JdbcTemplate을 사용하는 UserDao 메소드에서는 꼭 필요한 경우에만 DataAccessException을 잡아서 처리해주면 되고 그 외의 경우에는 무시해도 된다. 그래서 DAO의 메소드에 모두 throws SQLExcetpion이 사라진 것이다.

그 밖에도 스프링 API 메소드에 정의되어 있는 대부분의 예외는 런타임 예외다. 따라서 발생 가능한 예외가 있다고 해서 이를 처리하도로 강제하지 않는다.

4-2 예외 전환
----------------------------------------------------------------------------
스프링의 JdbcTemplate이 던지는 DataAccessException은 일단 SQLException을 런타임 예외로 포장해주는 역할을 한다. 또한 SQLException에 담긴 다루기 힘든 상세한 예외정보를 의미있는 예외로 전환해서 추상화 하기 위한 용도로 사용된다.


JDBC의 한계

JDBC는 자바를 이용해 DB에 접근하는 방법을 추상화된 API 형태로 정의해놓고, 각 DB 업체가 JDBC 표준을 따라 만들어진 드라이버를 제공하게 해준다. 내부 구현은 DB마다 다르겠지만 JDBC의 Connection, Statement, ResultSet 등의
표준 인터페이스를 통해 그 기능을 제공해주기 때문에 자바 개발자들은 표준화된 JDBC의 API에만 익숙해지면 DB의 종류에 상관없이 일관된 방법으로 프로그램을 개발할 수 있다.
하지만, 표준화된 JDBC API가 DB 프로그램 개발 방법을 학습하는 부담을 줄여 주지만 DB를 자유롭게 변경해서 사용할 수 있는 유현한 코드까지 보장해주지는 못한다. 그 이유는 다음 2가지와 같다.

1. 비표준 SQL
JDBC 코드에서 사용하는 SQL은 어느정도 표준화된 언어이고 몇가지 표준 규칙이 있지만 DB마다 표쥰을 따르지 않는 비표준 문법과 기능을 제공한다.
이런 비표준 문법을 특별한 기능을 사용하거나 성능을 향상시키기 위해 사용하면 결국 SQL query가 DAO 코드에 들어가게 되고 해당 DAO는 특정 DB에 종속적인 코드가 된다.
해결방안으로는 표준 SQL만 사용하는 방법, DB별로 별도의 DAO를 만드는방법, SQL을 외부에 독립시켜서 DB에 따라 변경해 사용하는 방법이 있다 (7장에서 살펴본다)

2. 호환성 엇는 SQLException의 DB 에러정보
DB마다 SQL만 다른게 아니라 에러의 종류와 원인도 제각각이기 때문에 JDBC는 데이터 처리 중에 발생하는 다양한 예외를 그냥 SQLException 하나에 모두 담아버린다. 
예외가 발생한 원인은 SQLException 안에 담긴 에러 코드와 SQL 상태정보를 참조해봐야 한다. 그런데 SQLException의 getErrorCode()로 가져올 수있는 DB에러코드는 DB별로 모두 다르다.
그래서 SQLException은 예외가 발생했을 때의 DB상태를 담은 SQL 상태정보를 getSQLState()로 추가 제공한다. SQLException이 이러한 상태 코드를 제공하는 이유는 DB에 독립적인 에러정보를 얻기 위해서다.
하지만, DB의 JDBC 드라이버에서 SQLException에 담을 상태 코드를 정확하게 만들어주지 않는다. 결과적으로 getSQLState()의 상태정보를 믿고 결과를 파악하도록 코드를 작성하는것은 위험하다.


DB 에러 코드 매핑을 통한 전환

SQL 상태 코드는 JDBC 드라이버를 만들 때 들어가는 것이므로 같은 DB라고 하더라도 드라이버를 만들 때마다 달라지기도 하지만, DB 에러코드는 어느정도 일관성이 유지된다.
DB별 에러 코드를 참고해서 발생한 예외의 원인이 무엇인지 해석해주는 기능을 만들자, 
스프링은 DataAccessException이라는 SQLException을 대체할 수 있는 런타임 예외를 정의하고 있을 뿐 아니라 DataAccessException의 서브클래스로 세분화된 예외 클래스들을 정의하고 있다. 
스프링은 DB별 에러 코드를 분류해서 스프링이 정의한 예외 클래스와 매핑해놓은 에러 코드 매핑정보 테이블을 만들어두고 이를 이용한다.
JdbcTemplate은 SQLException을 단지 런타임 예외임 DataAccessException으로 포장하는 것이 아니라 DB의 에러 코드를 DataAccessException 게층구의 클래스 중 하나로 매핑해준다.
JdbcTemplate에서 던지는 예외는 모두 DataAccessException의 서브클래스로, 드라이버나 DB메타정보를 참고해서 DB 종류를 확인하고 DB별로 미리 준비된 매핑정보를 참고해서 적절한 예외 클래스를 선택하기 떄문에 DB가 달라져도 같은 종류의 에러라면 동일한 예외를 받을 수 있다.


DAO 인터페이스와 DataAccessException 계층구조

DataAcessException은 JDBC의 SQLException을 전환하는 용도로만 쓰이는것은 아니다. JDBC 외의 자바 데이터 액세스 기술(JPA, JPO, iBatis 등등..)에서 발생하는 예외에도 적용된다.
DataAcessException은 의미가 같은 예외라면 데이터 액세스 기술의 종류와 상관없이 일관된 예외가 발생하도록 만들어준다.


DAO 인터페이스와 구현의 분리

DAO를 따로 만들어서 사용하는 이유? => 데이터 액세스 로직을 담은 코드를 성격이 다른 코드에서 분리하기 위하여 + 분리된 DAO는 전략패턴을 적용해 구현 방법을 변경해서 사용할 수 있게 만들기 위하여

DAO의 사용 기술과 구현 코드는 전략 패턴과 DI를 통해 DAO를 사용하는 클라이언트에게 감출 수 있지만, 메소드 선언에 나타나는 예외정보가 문제가 될 수 있다.
UserDao의 인터페이스를 분리해서 기술에 독립적인 인터페이스로 만들려면 다음과 같이 정의해야 한다.
```java
public interface UserDao {
  public void add(User user);
}
```
하지만 DAO에서 사용하는 데이터 액세스 기술의 API가 예외를 던지기 때문에 위와 같이 사용할 수 없다. 만약 JDBC API를 사용하는 UserDao 구현 클래스의 add 메소드라면 SQLException을 던질 것이다.
따라서 인터페이스 메소드도 다음과 같이 선언되어야 한다.
```java
public interface UserDao {
  public void add(User user) throws SQLException;
}
```
이렇게 정의한 인터페이스는 JDBC가 아닌 다른 데이터 엑세스 기술로 DAO 구현을 전환하면 각각의 데이터 엑세스 기술의 API는 독자적인 예외를 던지기 때문에 사용할 수 없다. 
```java
public void add(User user) throws Exception;
```
위와 같이 선언하면 간단하지만 무책임한 선언이다. 
다행히 JDBC보다는 늦게 등장한 JDO, Hibernate, JPA 등의 기술은 SQLException 같은 체크 예외 대신 런타임 예외를 사용한다. (throws 선언을 해주지 않아도 된다.)
SQLException을 던지는 JDBC API를 직접 사용하는 DAO가 문제인데, 이 경우 DAO 메소드 내에서 런타임 예외로 포장해서 던져줄 수 있고, JDBC를 이용한 DAO에서 모든 SQLException을 런타임 예외로 포장한다면, DAO 메소드는 처음 의도했던 대로 선언해도 된다.
UserDao의 인터페이스를 분리해서 기술에 독립적인 인터페이스로 만들려면 다음과 같이 정의해야 한다.
```java
public interface UserDao {
  public void add(User user);
}
```
이제 DAO에서 사용하는 기술에 완전히 독립적인 인터페이스 선언이 가능해졌다.
하지만 대부분의 데이터 엑세스 예외는 어플리케이션에서 복구 불가능하거나 할 필요가 없지만 다 무시해야하는건 아니다.
문제점은 데이터 엑세스 기술이 다르면 같은 원인으로도 다른 종류의 예외가 던져진다는 것이다. 
따라서 DAO를 사용하는 클라이언트 입장에서는 DAO의 사용기술에 따라서 예외 처리방법이 달라져야 한다. 결국 클라이언트가 DAO 기술에 의존적이 될 수 밖에 없다.


데이터 액세스 예외 추상화와 DataAccessException 계층구조

그래서 스프링은 자바의 다양한 데이터 액세스 기술을 사용할 때 발생하는 예외들을 추상화해서 DataAccessException 계층구조 안에 정리해놓앗다.
DataAccessException은 자바의 주요 데이터 액세스 기술에서 발생할 수 있는 대부분의 예외를 추상화하고 있다.
DataAccessException은 일부 기술에서만 공통적으로 나타나는 예외를 포함해서 대부분의 데이터 액세스 기술에서 발생할 수 있는 예외를 계층구조로 분류해놓았다.

=> JdbcTemplate과 같이 스프링의 데이터 액세스 지원 기술을 이용해 DAO를 만들면 사용 기술에 독립적인 일관성 있는 예외를 던질 수 있다. 결국 인터페이스 사용, 런타임 예외 전환과 함께 DataAccessException 예외 추상화를
적용하면 데이터 액세스 기술과 구현 방법에 독립적인 이상적인 DAO 를 만들 수가 있다.


기술에 독립적인 UserDao 만들기

인터페이스 적용

사용자 처리 DAO 인터페이스의 이름을 UserDao라 하고, Jdbc를 이용해 구현한 클래스 이름은 UserDaoJdbc라 하자, (추후에 JPA, Hibernate 등 다른 데이터 액세스 기술을 사용해 구현 클래스를 만든다면 UserDaoJpa, UserDaoHibernate 등으로 한다)
UserDao 인터페이스에는 UserDao 클래스에서 DAO의 기능을 사용하려는 클라이언트들이 필요한 것만 추출해 낸다.
```java
public interface UserDao {
  void add(User user);
  User get(String id);
  List<User> getAll();
  void deleteAll();
  int getCount();
  // setDataSource는 인터페이스에 포함하지 않는다, 구현 방법에 따라 변경될수 있고 UserDao를 사용하는 클라이언트가 알고 있을 필요도 없다.
}
```

UserDaoJdbc 클래스와 빈 클래스를 아래와 같이 변경한다
```xml
<bean id="userDao" class="com.example.demo.user.dao.UserDaoJdbc">
  <property name="dataSource" ref="dataSource"/>
</bean>
```
```java
public class UserDaoJdbc implements UserDao {
```

테스트 보완

UserDaoTest의 UserDao 인스턴스 변수 선언은 UserDaoJdbc로 바꿀 필요는 없다. 왜냐하면 @Autowired가 스프링의 컨텍스트 내에서 정의된 빈 중에서 인스턴스에 주입 가능한 타입의 빈을 찾아줄 것이고 
UserDao는 UserDaoJdbc가 구현한 인터페이스이기 때문에 UserDaoTest의 UserDao dao에 UserDaoJdbc 클래스로 정의된 빈을 넣는데 문제가 없다.
경우에 따라 의도적으로 UserDaoJdbc dao 라고 선언할 수도 있으나, UserDao 테스트는 DAO의 기능을 검증하는 것이 목표이지 JDBC를 이용한 구현에 관심이 있는게 아니기 때문에 UserDao라는 변수타입을 그대로 둬도 된다.

(그림 4-4 참조)

```java
@Test(expected=DataAccessException.class)
public void duplicateKey() {
  dao.deleteAll();

  dao.add(user1);
  dao.add(user1); // 이때 스프링의 DataAccessException 예외 중 하나가 던져져야 한다. 어떤 예외가 발생했는지 확인해보려면 (expected=DataAccessException.class)를 지우면 구체적인 하위 계층 예외클래스(DuplicatedKeyException)가 드러난다.
}
```

DataAccessException 활용 시 주의사항

스프링을 활용하면 DB종류나 데이터 엑세스 기술에 상관없이 키 값이 중복이 되는 상황에서는 동일한 예외가 발생하리라고 기대할 것이다. 하지만 DuplicatedKeyException은 JDBC를 이용하는 경우에만 발생한다.
DataAccessException이 데이터 액세스 기술에 상관없이 어느정도 추상화된 공통 예외로 변환해주긴 하지만 근본적인 한계 때문에 완벽하다고 기대할 수는 없다.
따라서 DataAccessException을 잡아서 처리하는 코드를 만드려면 미리 학습 테스트를 통해 실제로 전환되는 예외의 종류를 확인해봐야 한다.
만약 DAO에서 사용하는 기술의 종류와 상관없이 동일한 예외를 얻고 싶다면 DuplicatedUserIdExceptio 처럼 직접 예외를 정의해두고 각 DAO의 add메소드에서 좀 더 상세한 예외 전환을 해줘야 한다.
하이버네이트 예외의 경우라도 중첩된 예외로 SQLException이 전달되기 떄문에 이를 다시 스프링의 JDBC 예외 전환 클래스의 도움을 받아 처리할 수 있다.

SQLException을 직접 해석해 DataAccessException으로 변환하는 코드의 사용법을 알아보자
SQLException을 DataAccessException으로 코드에서 직접 전환하고 싶다면 SQLExceptionTranslator 인터페이스를 구현한 클래스 중 SQLErrorCodeSQLExceptoinTranslator를 사용하면 된다. 
이는 에러코드 변환에 필요한 DB의 종류를 알아내기 위해 현재 연결된 DataSource를 필요로 한다.

```java
public class UserDaoTest {
    @Autowired
    UserDao dao;
    
    @Autowired
    DataSource dataSource;
    
    ...
    
    @Test
    public void sqlExceptionTranslate() {
      dao.deleteAll();
    
      try {
        dao.add(user1);
        dao.add(user1);
      } catch(DuplicateKeyException ex) {
        SQLException sqlEx = (SQLException)ex.getRootCause(); // DuplicatedKeyException은 중첩된 예외로 SQLException을 가지고 있다.
        SQLExceptionTranslator set = new SQLErrorCodeSQLExceptionTranslator(this.dataSource); //코드를 이용한 SQLException 전환
        assertThat(set.translate(null, null, sqlEx), is(DuplicateKeyException.class)); //에러 메세지 만들 때 사용하는 정보이므로 null로 넣어도 된다.
      }
    }
}
```

4-3 정리
----------------------------------------------------------------------------
   * 예외를 잡아서 아무런 조치를 취하지 않거나 의미 없는 throws 선언을 남발하는 것은 위험하다
   * 예외는 복구하거나 예외처리 오브젝트로 의도적으로 전달하거나 적절한 예외로 전환해야 한다.
   * 좀 더 의미있는 예외로 변경하거나, 불필요한 catch/throws를 피하기 위해 런타임 예외로 포장하는 두 가지 방법의 예외 전환이 있다.
   * 복구할 수 없는 예외는 가능한 빨리 런타임 예외로 전환하는 것이 좋다.
   * 어플리케이션의 로직을 담기 위한 예외(어플리케이션 예외)는 체크 예외로 만든다.
   * JDBC의 SQLException은 대부분 복구할 수 없는 예외이므로 런타임 예외로 포장해야 한다.
   * SQLException의 에러 코드는 DB에 종속되기 떄문에 DB에 독립적인 예외로 전환될 필요가 있다.
   * 스프링은 DataAcessException을 통해 DB에 독립적으로 적용 가능한 추상화된 런타임 예외 계층을 제공한다
   * DAO를 데이터 액세스 기술에서 독립시키려면 인터페이스 도입과 런타임 예외 전환, 기술에 독립적인 추상화된 예외로 전환이 필요하다.
   