# Chapter 03 탬플릿

탬플릿이란? 바뀌는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로 독립시켜서 효과적으로 활용할 수 있도록 하는 방법

3-1 다시 보는 초난감 DAO
----------------------------------------------------------------------------
1장에서 개선해온 UserDao코드에 남아있는 문제점? => 예외상황에 대한 처리

정상적인 JDBC 코드의 흐름을 따르지 않고 어떤 이유로든 예외가 발생했을 경우에대 사용한 리소스를 반드시 반환하도록 개선해야한다.

```java
public void deleteAll() throws SQLException {

  Connection c = dataSource.getConnection();
  PreparedStatement ps = c.prepareStatement("delete from users");

  ps.executeUpdate();

  ps.close();
  c.close();
}
```

문제점 : Connection & PreparedSatement 리소스를 사용하는데, 각각 close 메소드가 실행되기 전 예외가 발생하면 자원반환하지 못한다.

해결법: try, catch, finally 를 통해 자원을 반환하자. (아래코드 참조, finally 블록을 이용해 자원 획득 역순으로 null check 후 지원반환, close에서도 SQLException 발생할 수 있으나 별달리 처리할 수 있는것이 없음)

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

JDBC 조회 기능의 예외처리

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
      } catch (SQLException e){
      }
    }
  }
}
```

=> 삭제기능의 예외처리에서 ResultSet에 대한 자원반환 예외처리만 추가되었음

3-2 변하는 것과 변하지 않는 것
----------------------------------------------------------------------------

3-1 코드의 문제점 : 유사한 기능을 개발하기 위해 재사용이 불가능하고 Copy & Patse 를 반복하다보면 실수할 수 있고, 낙관적으로 실수하지 않는다고 가정하더라도 변경이 발생할 시 중복된 코드 모두 수정해야한다.

3-1 코드의 개선방안 : 변하지 않지만 많은곳에서 중복되는 코드와, 로직에 따라 확장되거나 변경되는 코드를 잘 분리해야 한다.

분리와 재사용을 위한 디자인 패턴 적용하기 위해 리스트 3-1에서 변하는 부분과 변하지 않는 부분을 분리해보자.

```java
public void deleteAll() throws SQLException {

  Connection c = null;
  PreparedStatement ps = null;

  try {
    c = dataSource.getConnection();

    ps = c.prepareStatement("delete from users"); => 기능에 따라 변하는 부분

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

- 메소드 추출

먼저 생각해 볼 수 있는 방법은 변하는 부분을 메소드로 추출하는 것이다.

```java
public void deleteAll() throws SQLException {
  Connection c = null;
  PreparedStatement ps = null;
  try {
    c = dataSource.getConnection();
    ps = makeStatement(c);
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

private PreparedStatement makeStatement(Connection c) throws SQLException {
  PreparedStatement ps;
  ps = c.prepareStatement("delete from users");
  return ps;
}
```

=> 변하는 부분을 메소드로 독립시킨 결과, 분리된 메소드는 변하는 부분이기 때문에 재사용 가능하지 않아 큰 이득이 없어보인다.

- 탬플릿 메소드 패턴 적용

탬플릿 메소드 패턴은 상속을 통해 기능을 확장해서 사용하는 패턴으로, 변하지 않는 부분은 슈퍼클래스에 변하는 부분은 추상클래스에 두고 서브 클래스에서 변하는 부분을 오버라이드 하여 사용하게 한다.

```java
public abstract class UserDao {
 abstract protected PreparedStatement makeStatement(Connection c) throws SQLException;
}

public class UserDaoDeleteAll extends UserDao {
  protected PreparedStatement makeStatement(Connection c) throws SQLException {
    PreparedStatement ps = c.prepareStatement("delete from users");
    return ps;
  }
}
```
=> UserDao 클래스의 기능을 확장하고 싶을 때마다 상속을 통해 자유롭게 확장할 수 있고, 확장 떄문에 상위 클래스에 불필요한 변화는 생기지 않으니 OCP 원칙을 잘 따른다고 할 수 있다.

문제점 : DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 한다, 확장구조가 이미 클래스를 설계하는 시점에서 고정되어 버린다 (변하지 않는 JDBC try, catch, finally 블록을 가진 UserDao와 변하는 PreparedStatement를 담고있는 서브클래스들의 관계까 컴파일 시점에 이미 결정되어 있다.)

- 전략 패턴의 적용

OCP를 잘 지키는 구조이면서도 탬플릿 메소드 패턴보다 유연하고 확장성 뛰어난 것이 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 전략 패턴이다.
(책 그림 3-2)
Context의 contextMethod()에서 일정한 구조를 가지고 동작하다가 특정 확장 기능은 Strategy 인터페이스를 통해 외부에 독립된 전략 클래스에 위임하는 전략이다.

deleteAll 메소드의 컨텍스트와 변하는 전략은 다음과 같다.

  - DB 커넥션 가져오기
  - PreparedStatement를 만들어줄 외부 기능 호출 ===> 변할 수 있는 전략
  - 전달받은 PreparedStatement 실행
  - 예외가 발생하면 이를 다시 메소드 밖으로 던지기
  - 모든 경우에 만들어진 PreparedStatement와 Connection을 적절히 닫아주기
  
전략 패턴의 구조를 따라 이 기능을 인터페이스로 만들어두고 인터페이스의 메소드를 통해 PreparedStatement 생성 전략을 호출해주면 된다.

전략 인터페이스 & 전략 인터페이스를 상속한 실제 전략
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

전략 패턴을 따라 DeleteAllStatement 전략을 사용하는 deleteAll 컨텍스트
```java
public void deleteAll() throws SQLException {
  Connection c = null;
  PreparedStatement ps = null;
  try {
    c = dataSource.getConnection();
    StatementStrategy strategy = new DeleteAllStatement();
    ps = strategy.makePreparedStatement(c);
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

문제점 : 컨텍스트가 StatementStrategy 인터페이스뿐 아니라 특정 구현 클래스인 DeleteAllStatement를 직접 아고 있기 때문에 OCP를 완전히 충족시킨다고 보기 어렵다.

- DI 적용을 위한 클라이언트/컨텍스트 분리

전략 패턴에 따르면 컨텍스트가 어떤 전략을 사용할 것인가 결정하는 것은 컨텍스트가 아니라 컨텍스트를 사용하는 클라이언트 코드인것이 일반적이다.
클라이언트가 구체적인 전략중 하나를 선택하여 오브젝트로 만들어서 컨텍스트에 전달해주면 컨텍스트는 전달받은 전략을 실행할 뿐이다.

(그림 3-3)

이 구조에서 전략 오브젝트 생성과 컨텍스트로의 전달을 담당하는 책임을 분리시킨것이 바로 ObjectFactory 이며, 이를 일반화 한 것이 앞에서 살펴보았던 DI이다. 결국 DI란 이러한 전략패턴의 장점을 쉽게 활용할 수 있도록 만든 구조이다.
그림 3-3 구조를 적용한 코드를 살펴보자

```java
클라이언트
public void deleteAll() throws SQLException {
  StatementStrategy st = new DeleteAllStatement();
  jdbcContextWithStatementStratgy(st);
}
=> 구체적인 전략 오브젝트를 생성하고 컨텍스트에게 전달 및 호출해주는 책임을 가진다.

컨텍스트
public void jdbcContextWithStatementStratgy(StatementStrategy stmt) throws SQLException {
  Connection c = null;
  PreparedStatement ps = null;
  try {
    c = dataSource.getConnection();
    ps = stmt.makePreparedStatement(c);
    ps.executeUpdate();
  } catch (SQLException e) {
    throw e;
  } finally {
    if (ps != null) { try { ps.close(); } catch (SQLException e) {}}
    if (c != null) { try { c.close(); } catch (SQLException e) {}}
  }
=> 클라이언트로 부터 StatementStrategy 타입의 전략 오브젝트를 제공받고 JDBC try, catch, finally 블록 실행, 제공받은 전략 오브젝트는 PreparedStatement 생성에 사용

전략인터페이스
public interface StatementStrategy {
  PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}

인터페이스를 상속한 실제 전략
public class DeleteAllStatement implements StatementStrategy {
  public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
    PreparedStatement ps = c.prepareStatement("delete from users");
    return ps;
  }
}
```

3-3 JDBC 전략 패턴의 최적화
----------------------------------------------------------------------------

- 전략 클래스의 추가 정보

add 기능을 구현하기 위해 AddStatement 라는 실제 전략 클래스를 만들기위해서 add할 user정보를 클라이언트로 받아와야한다. => 생성자를 통해 받는다.

```java
클라이언트
public void addUser(User user) throws SQLException {
  StatementStrategy st = new AddStatement(user);
  jdbcContextWithStatementStratgy(st);
}
```

- 전략과 클라이언트의 동거

문제점: DAO 메소드마다 새로운 StatementStrategy 클래스 파일이 너무 많이 늘어난다 + 부가정보를 전달반아햐하는 전략의 경우 오브젝트를 전달받는 생성자와 이를 저장해둘 인스턴스 변수를 따로 생성해야한다.

해결법 : 로컬클래스 

StatementStrategy 전략 클래스를 매번 독립된 파일로 만들지 말고 UserDao 클래스 클라이언트 메소드(deleteAll, add 등등..) 내부에 내부 클래스로 정의해버린다. 

```java
public void add(User user) throws ClassNotFoundException, SQLException {
  내부 클래스
  class AddStatement implements StatementStrategy {
    User user;

    public AddStatement(User user){
      this.user = user;
    }

    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
      PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
      ps.setString(1, user.getId());
      ps.setString(2, user.getName());
      ps.setString(3, user.getPassword());

      return ps;
    }
  }

  클라이언트 코드
  StatementStrategy st = new AddStatement(user);
  jdbcContextWithStatementStratgy(st);
}
```

클래스 파일의 숫자가 주는것 이외에도 로컬 클래스는 클래스가 내부 클래스이기 때문에 자신이 선언된 곳의 정보에 접근할 수 있다는 장점이 있다 => 추가 정보를 전달받기 위한 생성자나 인스턴스 변수가 필요없다.

```java
public void add(final User user) throws ClassNotFoundException, SQLException {
  class AddStatement implements StatementStrategy {
    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
      PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
      ps.setString(1, user.getId());
      ps.setString(2, user.getName());
      ps.setString(3, user.getPassword());

      return ps;
    }
  }

  StatementStrategy st = new AddStatement(); => user 넘겨주지 않아도 된다
  jdbcContextWithStatementStratgy(st);
}
```

익명 내부 클래스

- 이름을 갖지 않는 클래스
- 클래스 선언과 오브젝트 생성이 결합된 형태로 만들어짐
- 상속할 클래스나 구현할 인터페이스를 생성자 대신 사용하여 다음과 같은 형태로 사용 => `new 인터페이스이름() { 클래스 본문 }`
- 클래스 재사용이 없고, 구현한 인터페이스 타입으로만 사용할 경우 적용

=> AddStatement 클래스가 add 메소드 안에서만 사용할 목적으로 만들어졌기 때문에 더 간결하게 클래스 이름도 제거할 수 있다.

익명클래스를 적용한 코드

```java
public void add(final User user) throws ClassNotFoundException, SQLException {
  jdbcContextWithStatementStratgy(new StatementStrategy() {
    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
      PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
      ps.setString(1, user.getId());
      ps.setString(2, user.getName());
      ps.setString(3, user.getPassword());

      return ps;
    }
  });
}
```
```java
public void deleteAll() throws SQLException {
  jdbcContextWithStatementStratgy(
      new StatementStrategy() {
        public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
          return c.prepareStatement("delete from users");
        }
      }
  );
}
```

3-4 컨텍스트와 DI
----------------------------------------------------------------------------

jdbcContextWithStatementStrategy 메소드는 JDBC의 일반적인 작업 흐름을 담고 있기 때문에 다른 DAO에서도 사용 가능하다. => jdbcContextWithStatementStrategy() 를 UserDao 클래스 밖으로 독립시켜서 다른 DAO가 사용할 수 있도록 수정해보자

클래스 분리

- 분리해서 만들 클래스 이름은 jdbcContext라고 하자.
- jdbcConetext에 UserDao에 있던 컨텍스트 메소드인 jdbcContextWithStatementStrategy를 workWithStatementStrategy 라는 이름으로 변경한다.
- UserDao가 분리된 JdbcConext를 DI받아서 사용할 수 있게 한다.

=> DB Connection이 필요한 코드는 jdbcConext안에 있기 때문에 jdbcContext는 datasource가 필요하다. 따라서, jdbcContext가 DataSource에 의존하고 있으므로 DataSource 타입 빈을 DI받을 수 있도록 필드로 가지게 한다. 

```java
public class JdbcContext {
	private DataSource dataSource;

	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}

	public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;
		try {
			c = dataSource.getConnection();
			ps = stmt.makePreparedStatement(c);
			ps.executeUpdate();
		} catch (SQLException e) {
			throw e;
		} finally {
			if (ps != null) { try { ps.close(); } catch (SQLException e) {}}
			if (c != null) { try { c.close(); } catch (SQLException e) {}}
		}
	}
}

public class UserDao {
	private JdbcContext jdbcContext;

	public void setJdbcContext(JdbcContext jdbcContext) {
		this.jdbcContext = jdbcContext;
	}

	public void add(final User user) throws ClassNotFoundException, SQLException {
		this.jdbcContext.workWithStatementStrategy(new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
				ps.setString(1, user.getId());
				ps.setString(2, user.getName());
				ps.setString(3, user.getPassword());

				return ps;
			}
		});
	}

	public void deleteAll() throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
				new StatementStrategy() {
					public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
						return c.prepareStatement("delete from users");
					}
				}
		);
	}
}
```

빈 의존관계 변경

UserDao는 이제 jdbcContext에 의존하고 있다. 그런데 jdbcConext는 인터페이스가 아닌 구현 클래스이다. 스프링의 DI는 기본적으로 인터페이스르 사이에 두고 의존 클래스를 바꿔서 사용하게 하는것이 목적이다. 하지만 이 경우 jdbcContext는 그 자체로 독립적인
JDBC 컨텍스트를 제공해주는 서비스 오브젝트로서 의미가 있을 뿐이고 구현 방법이 바뀔 가능성은 거의 없다 (다른 ORM을 쓰지 않는 이상..)

따라서 인터페이스를 구현하도록 만들지 않았고, UserDao와 jdbcConext는 인터페이스를 사이에 두지 않고 DI를 적용하는 구조가 되었다.
(그림 3-4)

스프링의 빈 설정은 클래스 레벨이 아니라 런타임에 만들어지는 오브젝트 레벨의 의존관계에 따라 정의된다. 빈으로 정의되는 오브젝트 사이의 관계를 살펴보면 다음과 같다.
(그림 3-5)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="jdbcContext" class="com.example.demo.user.dao.JdbcContext">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <bean id="userDao" class="com.example.demo.user.dao.UserDao">
        <property name="dataSource" ref="dataSource"/>
        <property name="jdbcContext" ref="jdbcContext"/>
    </bean>

    <bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
        <property name="driverClass" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost/tobi?serverTimezone=UTC"/>
        <property name="username" value="root"/>
        <property name="password" value="springbook"/>
    </bean>
</beans>
```

jdbcContext의 특별한 DI

지금까지 적용했던 일반적인 DI에서는 클래스 레벨에서 구체적인 의존관계가 만들어지지 않도록 인터페이스를 사용했다. 덕분에 코드에서 직접 클래스를 사용하지 않아도 됐고, 설정을 변경하는 것만으로도 얼마든지 다양한 의존 오브젝트를 변경해서 사용할 수 있게 됐다.
그런데, JdbcConext와 UserDao 사이에는 인터페이스를 사용하지 않고 DI를 적용했다.(오브젝트 레벨이 아닌 클래스 레벨에서 의존관계가 설정되었다.) 비록 런타임에 DI로 외부에서 오브젝트를 주입해주긴 하지만 의존 오브젝트의 구현 클래스를 변경할 수 없다.


스프링 빈으로 DI

의존 관계 주입이라는 개념을 충실히 따르자면 인터페이스를 사이에 둬서 클래스 레벨에서는 의존관계가 고정되지 않게 하고 런타임 시에 의존할 오브젝트와의 관계를 다이나믹하게 주입해주는 것이 맞다. 즉, 인터페이스를 사용하지 않았다면 엄밀히 말해서는 온전한 DI라고 보긴 어렵다.
하지만, 스프링 DI는 넓게 보자면 객체의 생성과 관계설정에 대한 제어권한을 오브젝트에서 제거하고 외부로 위임했다는 IoC 개념을 포괄한다. 이런 관점에서 JdbcContext를 스프링을 이용해 UserDao 객체에서 사용하게 주입했다는 건 DI의 기본을 따르고 있다고 볼 수 있다.

JdbcContext를 UserDao와 DI 구조로 만들어야할 이유
1. JdbcContext가 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈이 되기 때문이다 => jdbcConext는 상태정보를 갖고 있지 않다 또한, dataSource라는 인스턴스 변수는 있지만 읽기전용이므로 jdbcConext가 싱글톤이 되는데 문제가 없다.
2. JdbcContext가 DI를 통해 다른 빈에 의존하고 있기 때문이다. => jdbcContext는 dataSource 프로퍼티를 통해 DataSource 오브젝트를 주입받게 되어 있다. DI를 위해서는 주입받는 오브젝트와 주입하는 오브젝트 모두 스프링 빈으로 등록되어 있어야 한다.

여기서 중요한 점은 인터페이스의 사용 여부이다. 인터페이스가 없다는 건 UserDao와 JdbcConext가 매우 긴밀한 관계를 가지고 강하게 결합되어있음을 의미한다.
jdbcConext는 ORM을 교체하지 않는 이상 교체될 필요가 없다. 이런 경우에는 굳이 인터페이스를 두지 않고 강력한 결합을 가진 관계를 허용하면서 1. 싱글톤으로 만드는 것 2. DI를 위해 스프링 빈으로 등록해서 UserDao에 DI되도록 만들어도 좋다.


코드를 이용하는 수동 DI

jdbcContext를 스프링 빈으로 등록해서 UserDao에 DI 하는 대신 사용할 수 있는 방법 => UserDao 내부에서 직접 DI 적용

1. JdbcContext를 싱글톤으로 만드는 것은 포기 해야함. => DAO 메소드가 호출될 때마다 JdbcConext 오브젝트를 새로 만드는 방법으 아니고 DAO당 하나씩 JdbcConext를 가지고 있게 한다.
JdbcContext를 스프링 빈으로 등록하지 않았으므로 다른 누군가가 JdbcConext의 생성과 초기화를 책임져야한다. JdbcContext의 제어권은 UserDao가 갖는 것이 적당하다. 자신이 사용할 오브젝트를 직접 만들고 초기화 하는 방법을 사용하는 것
2. JdbcContext가 DI를 통해 다른 빈에 의존하고 있다 => 여전히 JdbcConext는 DataSource 타입을 다이나믹하게 주입받아야 한다. 그렇지 않으면 DataSource 구현 클래스를 자유롭게 바꿔가면서 적용할 수 없다. => UserDao에게 JdbcConext의 DI까지 위임하자!
JdbcConext에게 주입해줄 DataSource를 UserDao가 스프링 컨테이너로부터 주입받아 JdbcContext에게 주입해준다. (그림 3-6)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userDao" class="com.example.demo.user.dao.UserDao">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
        <property name="driverClass" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost/tobi?serverTimezone=UTC"/>
        <property name="username" value="root"/>
        <property name="password" value="springbook"/>
    </bean>
</beans>
```
```java
public class UserDao {
  private DataSource dataSource;
  private JdbcContext jdbcContext;

  public void setDataSource(DataSource dataSource) {
    // jdbcContext 직접 생성(IoC)
    this.jdbcContext = new JdbcContext();
    // 직접 생성한 jdbcConext에 주입받은 dataSource 의존 오브젝트 주입(DI)
    this.jdbcContext.setDataSource(dataSource);
    this.dataSource = dataSource;
  }
  ...
```

이 방법의 장점은 굳이 인터페이스를 두지 않아도 될 만큼 긴밀한 관계를 갖는 DAO 클래스와 JdbcContext를 어색하게 따로 빈으로 분리하지 않고 내부에서 직접 만들어 사용하면서도 다른 오브젝트에 대한 DI를 적용할 수 있다는 점이다.

인터페이스를 사용하지 않는 클래스와의 의존관계지만 스프링의 DI를 이요하기 위해 빈으로 등록해서 사용하는 방법
 - 장점: 오브젝트 사이의 실제 의존관계가 설정파일에 명확하게 표현됨
 - 단점: DI의 원칙에 부합하지 않는 구체적인 클래스와의 관계가 설정에 노출됨

DAO 코드를 이용해 수동으로 DI하는 방법
 - 장점: JdbcContext가 UserDao의 내부에서 만들어지고 사용되는 관계가 외부에 드러나지 않음
 - 단점: 싱글톤으로 만들 수 없고, DI작업을 위한 추가코드가 필요 (setDataSource)
 
 3-5 템플릿과 콜백
 ---------------------------------------------------------------------------

앞서 살펴본 구조는 복잡하지만 바뀌지 않는 일정한 패턴을 갖는 작업흐름이 존재하고 그 중 일부만 바꿔서 사용해야 하는 경우에 적합한 전략패턴 구조에 익명 내부클래스를 조합 하였음

이러한 방식을 탬플릿/콜백 패턴이라고 부른다.

탬플릿: 어떤 목적을 위해 미리 만들어둔 틀 (전략 패턴의 컨텍스트) / 콟백: 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달되는 오브젝트 (익명 내부 클래스로 만들어지는 오브젝트)

템플릿/콜백의 특징
 - 일반적으로 단일 메소드 인터페이스를 사용한다 => 템플릿의 작업 흐름 중 특정 기능을 위해 한 번 호출되는 경우가 일반적이기 때문
 - 콜백 인터페이스의 메소드는 보통 파라미터가 있고, 템플릿의 컨텍스트 정보를 받을 때 사용된다 (예를 들어 connection 객체)
 - (그림 3-7) 
 - 클라이언트의 역할은 템플릿 안에서 실행될 로직을 담은 콜백 오브젝트를 만들고, 콜백이 참조할 정보(파라미터)를 제공하는 것이다. 만들어진 콜백은 클라이언트가 탬플릿의 메소드를 호출할때 파라미터로 제공된다.
 - 템플릿은 정해진 작업 흐름을 따라 작업을 진행하다가 내부에서 생성한 참조정보를 가지고 콜백 오브젝트의 메소드를 호출한다. 콜백은 클라이언트 메소드에 있는 정보와 템플릿이 제공한 참조정보를 이용해서 작업을 수행하고 결과를 탬플릿에게 돌려준다.
 - 탬플릿은 콜백이 돌려준 정보를 사용해서 작업을 마저 수행한다. 경우에 따라 최종 결과를 클라이언트에 다시 돌려주기도 한다.
 
 일반적인 DI라면 템플릿에 인스턴스 변수를 만들어두고 사용할 의존 오브젝트를 setter로 받아 사용할 것이다. 반면에 템플릿/콜백 패턴에서는 매번 메소드 단위로 사용할 오브젝트를 새롭게 전달(수동 DI) 받는다는 것이 특징이다.
 콜백 오브젝트가 내부 클래스로서 자신을 생성한 클라이언트 메소드 내의 정보를 직접 참조한다는 것도 템플릿/콜백의 특징이다.
 
 템플릿/콜백 패턴의 아쉬운 점?
 => DAO 메소드에서 매번 익명 내부 클래스를 사용하기 때문에 상대적으로 코드를 작성하고 읽기가 불편하다. => 콜백을 분리하여 복잡한 익명 내부 클래스의 사용을 최소화하자.
 
 콜백의 분리와 재활용
 deleteAll 메소드와 익명 내부 클래스로 만든 콜백 오브젝트 구조를 살펴보면 고정된 SQL 쿼리를 담아서 PreparedStatement를 만들어 반환하는게 전부다.
 즉 바인딩할 파라미터 없이 미리 만들어진 SQL을 이용해 PreparedStatement를 만들기만 하기 때문에 SQL 문장만 파라미터로 받아서 바꿀 수 있게 하고 메소드 내용 전체를
 분리해 별도의 메소드로 만들어보자
 
 ```java
    public void deleteAll() throws SQLException {
        executeSql("delete from user");   
   } 

 	public void executeSql(final String query) throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
				new StatementStrategy() {
					public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
						return c.prepareStatement(query);
					}
				}
		);
	}
}
```
 
 바뀌지 않는 모든 부분을 빼내서 executeSql 메소드로 만들었다. 바뀌는 부분인 SQL 문장만 파라미터로 받아 사용하게 하여 재활용 가능한 콜백을 담은 메소드가 만들어졌다.
 이제 고정된 SQL을 실행하는 DAO 메소드는 deleteAll 메소드 처럼 executeSql 을 호출하는 한줄이면 끝이다.
 변하는 것과 변하지 않는 것을 분리하고 변하는 것은 유연하게 재활용 할 수 있게 만든다는 원리를 적용하면 단순하고 안전하게 작성 가능한 JDBC 활용 코드가 완성된다.

 
콜백과 템플릿의 결합
 
executeSql 메소드는 UserDao만 사용할 것이 아니라 DAO가 공유할 수 이는 템플릿 클래스 안으로 옮기는게 좋다.
엄밀히 말해서 템플릿은 jdbcContext 클래스가 아니라 workWithStatementStrategy 메소드이므로 jdbcContext 클래스로 콜백 생성(new StatementStrategy)과 템플릿 호출(workWithStatementStrategy)이 담긴 executeSql 메소드를 옮긴다고 해서 문제 될것이 없다.
 
```java
    public class JdbcContext {
        private DataSource dataSource;
    
        public void setDataSource(DataSource dataSource) {
            this.dataSource = dataSource;
        }
    
        public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
            Connection c = null;
            PreparedStatement ps = null;
            try {
                c = dataSource.getConnection();
                ps = stmt.makePreparedStatement(c);
                ps.executeUpdate();
            } catch (SQLException e) {
                throw e;
            } finally {
                if (ps != null) { try { ps.close(); } catch (SQLException e) {}}
                if (c != null) { try { c.close(); } catch (SQLException e) {}}
            }
        }
        
        public void executeSql(final String query) throws SQLException {
            workWithStatementStrategy(
                    new StatementStrategy() {
                        public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                            return c.prepareStatement(query);
                        }
                    }
            );
        }
    }      

    public class UserDao {
        public void deleteAll() throws SQLException {
            this.jdbcContext.executeSql("delete from user");   
       }
    }
 
```
이제 모든 DAO 메소드에서 executeSql 메소드를 사용할 수 있게 됐다. jdbcContext 안에 클라이언트, 템플릿, 콜백이 함께 공종하면서 동작하는 구조가 되었다.
(그림 3-9)

하나의 목적을 위해 서로 긴밀하게 연관되어 동작하는 응집력이 강한 코드들이기 때문에 한 군데 모여있는게 유리하다. 
구체적인 구현, 내부의 전략패턴, 코드에 의한 DI, 익명 내부 클래스 등의 기술은 최대한 감추고 외부에는 필요한 기능을 제공해주는 단순한 메소드만 노출한다.


템플릿/콜백의 응용

탬플릿/콜백 패턴은 스프링에서만 사용할 수 있다거나 스프링만이 제공해주는 독립적인 기술이 아니다. 하지만 스프링은 이 패턴을 적극적으로 활용한다.
DI도 순수한 스프링의 기술은 아니다. 스프링은 단지 이를 편리하게 사용할 수 있도록 도와주는 컨테이너를 제공하고 이런 패턴의 사용 방법을 지지해준다.

고정된 작업 흐름을 가지고 있으면서 자주 반복되는 코드가 있다면 중복되는 부분을 먼저 메소드로 분리해보자. 분리한 작업을 필요에따라 바꾸어 사용해야 한다면 인터페이스를 사이에 두고
분리해서 전략 패턴을 적용하고 DI로 의존관계를 관리하도록 만든다. 바뀌는 부분이 한 어플리케이션 안에서 동시에 여러 종류가 만들어질 수 있다면 템플릿/콜백 패턴을 적용하는것을 고려해볼 수 있다.

가장 전형적인 템플릿/콜백 패턴의 후보는 try/catch/finally 블록을 사용하는 코드이다.


테스트와 try/catch/finally

```java
public class CalcSumTest {
  @Test
  public void sumOfNumbers() throws IOException {
    Calculator calculator = new Calculator();
    int sum = calculator.calcSum(getClass().getResource("number.txt").getPath());

    assertThat(sum, is(10));
  }
}

public class Calculator {
  public int calcSum(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    Integer sum = 0;
    String line = null;

    while ((line = br.readLine()) != null) {
        sum += Integer.valueOf(line);
    }

    br.close();
    return sum;
  }
}
```
초난감 DAO와 마찬가지로 calcSum() 메소드도 파일을 읽거나 처리하다가 예외가 발생하면 파일이 정상적으로 닫히지 않고 메소드를 빠져나가는 문제가 발생한다.
=> try/catch/finally 블록을 적용해서 어떤 경우더라도 파일이 열려있으면 닫아주도록 처리해야한다.

```java
public class CalcSumTest {
  @Test
  public void sumOfNumbers() throws IOException {
    Calculator calculator = new Calculator();
    int sum = calculator.calcSum(getClass().getResource("number.txt").getPath());

    assertThat(sum, is(10));
  }
}

public class Calculator {
  public int calcSum(String filePath) throws IOException {
    BufferedReader br = null; 
    
    try {
        br = new BufferedReader(new FileReader(filePath));
        Integer sum = 0;
        String line = null;
        while ((line = br.readLine()) != null) {
            sum += Integer.valueOf(line);
        }
        return sum;
    } catch(IOException e) {
        System.out.println(e.getMessage());
    } 
    finally {
        if(br != null) {
            try {br.close();}
            catch(IOException e) { System.out.println(e.getMessage();) }
        }
    }
  }
}
```

만들어진 모든 리소스는 확실히 정리하고 빠져나오도록 만들었고, 모든 예외상황에 대해서는 적절한 처리를 해주도록 하였다.


중복 제거와 템플릿/콜백 설

파일에 있는 모든 숫자의 곱을 계산하는 기능이 추가로 필요하다 등등 파일을 읽어서 처리하는 비슷한 기능이 새로 필요할 때마다 앞에서만든 기능을 복사해서 사용하지 말고 텤플릿/콜백 패턴을 적용해보자

먼저, 템플릿에 담을 반복되는 작업 흐름은 어떤 것인지 살펴보자. 템플릿이 콜백에게 전달해줄 내부의 정보는 무엇이고, 콜백이 템플릿에게 돌려줄 내용은 무엇인지 생각해보자. 템플릿/콜백을 적용할 때는 템플릿과 콜백의 경계를 파악하고
템플릿이 콜백에게, 콜백이 템플릿에게 각각 전달하는 내용이 무엇인지 파악하는 게 가장 중요하다. 그에 따라 콜백의 인터페이스를 정의해야하기 때문이다.

가장 쉽게 생각해볼 수 있는 구조는 템플릿이 파일을 열고 각 라인을 읽어올 수 있는 BufferReader를 만들어서 콜백에게 전달해주고, 콜백이 각 라인을 읽어서 숫자를 계산한 뒤 최종 결과만 템플릿에게 주는 것이다.

```java
// BufferReader를 전달받는 콜백 인터페이스
public interface BufferedReaderCallback {
   Integer doSomethingWithReader(BufferedReader br) throws IOException;
}


public class Calculator {

  public Integer calcSum(String filepath) throws IOException {
    BufferedReaderCallback sumCallback = new BufferedReaderCallback() {
      public Integer doSomethingWithReader(BufferedReader br) throws IOException {
        Integer sum = 0;
        String line = null;

        while ((line = br.readLine()) != null) {
            sum += Integer.valueOf(line);
        }

        return sum;
      }
    };
    return fileReadTemplate(filepath, sumCallback);
  }
  
    public Integer calcMultiple(String filepath) throws IOException {
    BufferedReaderCallback multipleCallback = new BufferedReaderCallback() {
      public Integer doSomethingWithReader(BufferedReader br) throws IOException {
        Integer multifly = 1;
        String line = null;

        while ((line = br.readLine()) != null) {
            multifly *= Integer.valueOf(line);
        }

        return multifly;
      }
    };
    return fileReadTemplate(filepath, multipleCallback);
  }

  // BufferReaderCallback 인터페이스를 사용하는 템플릿 메소드
  public Integer fileReadTemplate(String path, BufferedReaderCallback callback) throws IOException {
    BufferedReader br = null;

    try {
      br = new BufferedReader(new FileReader(path));
      int ret = callback.doSomethingWithReader(br);
      return ret;
    } catch (IOException e) {
      e.printStackTrace();
      throw e;
    } finally {
      if (br != null) {
        try {
          br.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
  }
}
```

```java
public class CalcSumTest {
  Calculator calculator;
  String numFilePath;

  @Before
  public void setUp() {
    this.calculator = new Calculator();
    this.numFilePath = getClass().getResource("numbers.txt").getPath();
  }

  @Test
  public void sumOfNumbers() throws IOException { 
    assertThat(calculator.calcSum(this.numFilePath), is(10));
  }

  @Test
  public void multipleOfNumbers() throws IOException {    
    assertThat(calculator.calcMultiple(this.numFilePath), is(24));
  }
}
```

템플릿/콜백의 재설계

calcSum과  calcMultiply에 나오는 두개의 콜백을 비교해보면 계산하는 부분만 다르고 버퍼에서 라인을 한줄씩 읽는 부분은 동일하다. 
=> 템플릿/콜백 패턴을 적용하자

```java
// 콜백 인터페이스
public interface LineCallback {
  Integer doSomethingWithLine(String line, Integer value);
}
```
```java
public class Calculator {
  public int calcSum(String filepath) throws IOException {
    LineCallback sumCallback = new LineCallback() {
      @Override
      public Integer doSomethingWithLine(String line, Integer value) {
        return value + Integer.valueOf(line);
      }
    };
    return lineReadTemplate(filepath, sumCallback, 0);
  }

  public Integer calcMultiple(String filepath) throws IOException {
    LineCallback multipleCallback = new LineCallback() {
      @Override
      public Integer doSomethingWithLine(String line, Integer value) {
        return value * Integer.valueOf(line);
      }
    };
    return lineReadTemplate(filepath, multipleCallback, 1);
  }

  // 템플릿
  public Integer lineReadTemplate(String path, LineCallback callback, int initVal) throws IOException {
    BufferedReader br = null;

    try {
      br = new BufferedReader(new FileReader(path));
      Integer res = initVal;
      String line = null;
      while ((line = br.readLine()) != null) { // 파일의 각 라인을 루프를 돌면서 읽는 것도 템플릿이 담당한다.
          res = callback.doSomethingWithLine(line, res); // 콜백이 계산한 값을 템플릿이 저장해 뒀다가 다음 계산에 다시 사용한다 , 각 라인의 내용을 가지고 계산하는 작업만 콜백에 맡긴다.
      }
      return res;
    } catch (IOException e) {
      e.printStackTrace();
      throw e;
    } finally {
      if (br != null) {
        try {
          br.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
  }
}
```
=> 로우레벨의 파일 처리 코드가 템플릿으로 분리되고 순수한 계산 로직만 남아 있기 떄문에 코드의 관심이 무엇인지 명확하게 보인다.

템플릿/콜백 패턴은 다양한 작업에 손쉽게 활용할 수 있다. 콜백이라는 이름이 의미하는 것처럼 다시 불려지는 기능을 만들어서 보내고 템플릿과 콜백, 클라이언트 사이 정보를 주고받는 일이 복잡하게 느껴질 수 있지만 코드의 특성이 바뀌는 경계를 
잘 살펴 인터페이스로 분리하는 원칙에만 충실하면 쉽게 템플릿/콜백 패턴을 사용할 수 있다.


제네릭스를 이용한 콜백 인터페이스

만약 파일을 라인 단위로 처리해서 만드는 결과의 타입을 다양하게 가져가고 싶다면 자바의 타입 파라미터라는 개념을 도입한 제네릭스를 이용하면 된다.

```java
// 콜백 인터페이스
public interface LineCallback<T> {
  T doSomethingWithLine(String line, T value);
}
```
```java
// 템플릿
public <T> T lineReadTemplate(String filepath, LineCallback<T> callback, T initVal) throws IOException {
  BufferedReader br = null;
  try {
    br = new BufferedReader(new FileReader(filepath));
    T res = initVal;
    String line = null;
    while((line = br.readLine()) != null) {
      res = callback.doSomethingWithLine(line, res);
    }
    return res;
  }
  catch(IOException e) {...}
  finally {...}
}

public String concatenate(String filepath) throws IOException {
  LineCallback<String> concatCallback = new LineCallback<String>() {
    public String doSomethingWithLine(String line, String value) {
      return value + line;
    }
  };

  return lineReadTemplate(filepath, concatCallback, "");
}
```

=> 이렇게 범용적으로 만들어진 템플릿/콜백을 이용하면 파일을 라인 단위로 처리하는 다양한 기능을 편리하게 사용할 수 있다.


 3-6 스프링의 jdbcTemplate
 ---------------------------------------------------------------------------
 
 스프링이 제공하는 템플릿/콜백 기능을 알아보자
 스프링은 거의 모든 종류의 JDBC 코드에 사용 가능한 템플릿과 콜백을 제공하고 자주 사용되는 패턴을 가진 콜백은 다시 템플릿에 결합시켜서 간단한 메소드 호출만으로 사용 가능하게 만들어져 있다.
 
 스프링이 제공하는 JDBC 코드용 기본 템플릿은 JdbcTemplate이다.
 
 ```java
public class UserDao {
  private DataSource dataSource;
  private JdbcTemplate jdbcTemplate;

  public void setDataSource(DataSource dataSource) {
    this.jdbcTemplate = new JdbcTemplate();
    this.dataSource = dataSource;
  }
```

update()

deleteAll에 먼저 적용해보자. 
 
PreparedStatementCreator 콜백 
createPreparedStatement 콜백 메소드 
update 템플릿 메소드

 ```java
//jdbcTemplate 적용
public void deleteAll() throws SQLException {
  this.jdbcTemplate.update(new PreparedStatementCreator() {
    public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
      return con.prepareStatement("delete from users");
    }
  });
}
```

내장 콜백을 사용하면 쉽게 구현할 수 있다.

```java
// 내장 콜백을 사용하는 update 적용
public void deleteAll() throws SQLException {
  this.jdbcTemplate.update("delete from users");
}
```

SQL로 PraparedStatement를 만들고 함께 제공하는 파라미터를 순서대로 바인딩 해주는 update 메소드를 활용하면 add 기능도 쉽게 구현할 수 있다.
```java
public void add(final User user) throws ClassNotFoundException, SQLException {
  this.jdbcTemplate.update("insert into users(id, name, password) values (?,?,?)", user.getId(), user.getName(), user.getPassword());
}
```

queryForInt()

PreparedStatementCreator 콜백
createPreparedStatement 콜백 메소드
ResultSetExtractor 콜백
extractData 콜백 메소드
query 탬플릿 메소드

```java
public int getCount() throws SQLException {
  return this.jdbcTemplate.query(new PreparedStatementCreator() {
    public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
      return con.prepareStatement("select count(*) from users");
    }
  }, new ResultSetExtractor<Integer>() {
    public Integer extractData(ResultSet rs) throws SQLException, DataAccessException {
      rs.next();
      return rs.getInt(1);
    }
  });
}
```

역시 내장 콜백을 사용하면 쉽게 구현할 수 있다. (JdbcTemplate은 이런 기능을 가진 콜백을 내장하고 있는 queryForInt() 라는 편리한 메소드를 제공한다. Integer 타입의 결과를 가져올 수 있는 SQL 문장만 전달해주면 된다.)
```java
public void add(final User user) throws ClassNotFoundException, SQLException {
  this.jdbcTemplate.queryForInt("select count(*) from users");
}
```

jdbcTemplate은 스프링이 제공하는 클래스지만 DI 컨테이너를 굳이 필요로 하지 않는다. UserDao처럼 직접 JdbcTemplate 오브젝트를 생성하고 필요한 DataSource를 전달해주기만 하면 JdbcTemplate의 모든 기능을 사용할 수 있다.


queryForObject()

위 queryForInt의  getCount에 적용했던 ResultSetExtractor 콜백 대신 RowMapper콜백을 사용한다. ResultSetExtractor와 RowMapper 모두 템플릿으로 부터 ResultSet을 전달받고, 필요한 정보를 추출해서 리턴하는
방식으로 동작하지만 ResultSetExtractor는 ResultSet을 한번 전달받아 알아서 추출작업을 모두 진행하고 최종결과만 리턴해주는데 반해, RowMapper는 ResultSet의 로우 하나를 매핑하기 위해 여러번 호출될 수 있다.

RowMapper 콜백
mapRow 콜백 메소드
queryForObject 탬플릿 메소드

```java
public User get(String id) {
  return this.jdbcTemplate.queryForObject("select * from users where id = ?",
      new Object[]{id},
      new RowMapper<User>() {
        public User mapRow(ResultSet rs, int rowNum) throws SQLException {
          User user = new User();
          user.setId(rs.getString("id"));
          user.setName(rs.getString("name"));
          user.setPassword(rs.getString("password"));
          return user;
        }
  });
}
```

queryForObject는 SQL을 실행하면 한개의 로우만 얻을 것이라고 기대하고 ResultSet의 next를 실행하여 첫번째 로우로 이동한 다음 RowMapper 콟백을 호출한다.
RowMapper에서는 현재 ResultSet이 가르키고 있는 로우의 내용을 User 오브젝트에 담아 리턴해주기만 하면된다.
RowMapper가 리턴한 User 오브젝트는 queryForObject 메소드의 리턴값으로 get메소드에 전달된다.
queryForObject는 SQL을 실행해서 받은 로으의 개수가 1개가 아니라면 예외를 던진다.


query()

get 메소드는 하나의 로우을 User 오브젝트에 담았으니 모든 로으를 가져오는 getAll은 List<User> 를 반환타입으로 하고, PK인 id순으로 정렬하여 반환받자.

getAll 메소드가 만들어졌다고 가정하고, 검증하는 테스트 코드부터 짜면 아래와 같다.

```java
@Test
public void getAll() {
  dao.deleteAll();

  dao.add(user1);
  List<User> users1 = dao.getAll();
  assertEquals(users1.size(), 1);
  checkSameUser(user1, users1.get(0));

  dao.add(user2);
  List<User> users2 = dao.getAll();
  assertEquals(users1.size(), 2);
  checkSameUser(user1, users2.get(0));
  checkSameUser(user2, users2.get(1));

  dao.add(user3);
  List<User> users3 = dao.getAll();
  assertEquals(users3.size(), 3);
  checkSameUser(user3, users3.get(0));
  checkSameUser(user1, users3.get(1));
  checkSameUser(user2, users3.get(2));
}

private void checkSameUser(User user1, User user2){
  assertEquals(user1.getId(), user2.getId());
  assertEquals(user1.getName(), user2.getName());
  assertEquals(user1.getPassword(), user2.getPassword());
}
```

query() 템플릿을 이용하는 getAll 구현

jdbcTemplate의 query 메소드 사용한다.
queryForObject는 쿼리의 결과가 로우 하나일때 사용하고, query는 쿼리의 결과가 일반적인 로우 다수일때 사용한다. query의 리턴타입은 List<T> 이고, query의 제네릭 메소드 타입은
파라미터로 넘기는 RowMapper<T> 콜백 오브젝트에서 결정된다.

```java
public List<User> getAll() {
  return this.jdbcTemplate.query("select * from users order by id",
      new RowMapper<User>() {
        public User mapRow(ResultSet rs, int rowNum) throws SQLException {
          User user = new User();
          user.setId(rs.getString("id"));
          user.setName(rs.getString("name"));
          user.setPassword(rs.getString("password"));
          return user;
        }
      });
}
```

query의 첫번째 파라미터는 SQL 쿼리이고, 두번째 파라미터는 바인딩할 파라미터이다(없다면 생략), 마지막 파라미터는 RowMapper 콜백이다. 
query 템플릿은 SQL을 실행해서 얻은 ResultSet의 모든 로우르르 열람하면서 로우마다 RowMapper를 호출한다. SQL 쿼리를 실행해 DB에서 가져오는 로우의 개수만큼 RowMapper 콟백이 호출된다.
RowMapper는 현재 로우의 내용을 User타입 오브젝트에 매핑해서 돌려주고, 이는 템플릿이 미리 준비해해둔 List<User>에 추가된다. 모든 로우에 대한 작업을 마치면 List<User>가 반환된다.


테스트 보완

네거티브 테스트라 불리는 예외 상황에 대한 테스트는 빼먹기 쉽지만 이런 케이스에 대한 테스트까지 해야한다.

getAll의 쿼리를 실행했는데 아무런 데이터가 없는경우? 
=> getAll은 일단 query라는 템플릿을 사용했으니 query가 이런경우 어떤 결과를 반환해주는지 알아보자. queryForObject처럼 예외를 던지지는 않고 크기가0인 List<T>를 반환한다.

```java
List<User> users0 = dao.getAll();
assertEquals(users0.size(), 0);
```

getAll이 내부적으로 query 템플릿을 사용했다고 해서 아무런 데이터 없이 반환될 경우 size가 0인 List를 반환할 필요는 없다. 경우에 따라 null 또는 예외를 던지게 할 수도 있다.


재사용 가능한 콜백의 분리

DI를 위한 코드정리

UserDao의 모든 메소드가 JdbcTemplate을 이용하도록 수정했으니 DataSource를 직접 사용할 일은 없다. 단지 JdbcTemplate을 생성하면서 직접 DI 해주기 위해 필요한 DataSource를
전달받아야하니 setter는 남긴다.
```java
private JdbcTemplate jdbcTemplate;

public void setDataSource(DataSource dataSource) {
  this.jdbcTemplate = new JdbcTemplate(dataSource);
}
```

중복 제거

get과 getAll을 보면 RowMapper의 내용이 같다. 
=> 재사용가능성과 확장성을 확보하기 위해 RowMapper 콜백을 분리해서 중복을 없애고 재사용 가능하게 만들자, RowMapper에는 상태정보가 없기 때문에 하나의 콜백 오브젝트를 멀티 스레드에서 공유해도 괜찮다.
```java
private RowMapper<User> userMapper = new RowMapper<User>() {
  public User mapRow(ResultSet rs, int rowNum) throws SQLException {
    User user = new User();
    user.setId(rs.getString("id"));
    user.setName(rs.getString("name"));
    user.setPassword(rs.getString("password"));
    return user;
  }
};

public User get(String id) {
  return this.jdbcTemplate.queryForObject("select * from users where id = ?",
      new Object[]{id}, this.userMapper);
}

public List<User> getAll() {
  return this.jdbcTemplate.query("select * from users order by id",this.userMapper);
}
```

템플릿/콜백 패턴과 UserDao

3장에서 템플릿/콜백 패턴을 학습하면서 개선을 거듭한 최종 UserDao 코드를 살펴보자.
```java
public class UserDao {
  private RowMapper<User> userMapper = new RowMapper<User>() {
    public User mapRow(ResultSet rs, int rowNum) throws SQLException {
      User user = new User();
      user.setId(rs.getString("id"));
      user.setName(rs.getString("name"));
      user.setPassword(rs.getString("password"));
      return user;
    }
  };

  private JdbcTemplate jdbcTemplate;

  public void setDataSource(DataSource dataSource) {
    this.jdbcTemplate = new JdbcTemplate();
  }

  public void add(final User user) throws ClassNotFoundException, SQLException {
    this.jdbcTemplate.update("insert into users(id, name, password) values (?,?,?)", user.getId(), user.getName(), user.getPassword());
  }

  public User get(String id) {
    return this.jdbcTemplate.queryForObject("select * from users where id = ?",
        new Object[]{id}, this.userMapper);
  }

  public void deleteAll() throws SQLException {
    this.jdbcTemplate.update("delete from users");
  }

  public int getCount() throws SQLException {
    return this.jdbcTemplate.queryForObject("select count(*) from users", Integer.class);
  }

  public List<User> getAll() {
    return this.jdbcTemplate.query("select * from users order by id",this.userMapper);
  }
}
```

UserDao 에는 User 정보를 DB에 생성/수정/삭제 하는 방법에 대한 로직만 담거있다.
User라는 java object와 USER 테이블 사이에 어떻게 정보를 주고 받을지, DB와 커뮤니케이션 하기 위한 SQL 문장이 어떤 것인지에 대한 최적화된 코드를 가지고 있다.
만약 사용할 테이블과 필드정보가 바뀌면 UserDao의 거의 모든 코드가 바뀌기 때문에 높은 응집도를 가지고 있다.

JdbcTemplate에는 JDBC API를 사용하는 방식, 예외처리, 리소스 반납, DB connection 획득 등 에 대한 관심과 책임이 모두 담겨있다.
따라서 변경이 일어난다고 해도 UserDao 코드에는 영향을 주지 않는다. 그런 면에서 책임이 다른 코드와 낮은 결합도를 유지하고 있다고 볼 수 있다.

다만, JdbcTemplate이라는 템플릿 클래스를 직접 이용한다는 면에서 특정 템플릿/콜백 구현에 대한 강한 결합을 가지고 있다. 하지만, JdbcTemplate이 스프링에서
JDBC를 이용하여 DAO를 만드는 사실상 표준 기술이고, JDBC대신 다른 데이터 액세스 기술을 사용하지 않는 한 바뀔리도 없겠지만, 그래도 더 낮은 결합도를 유지하고 싶다면 JdbcTempalte을
독립적인 빈으로 등록하고 JdbcTemplate이 구현하공 있는 JdbcOperations 인터페이스를 통해 DI 받아 사용하면 된다.

개선사항?
 - userMapper가 인스턴스 변수로 설정되어 있고, 한번 만들어지면 변경되지 않는 프로퍼티와같은 성격을 띄고 있으니, 아예 UserDao 빈의 DI용 프로퍼티로 만들수 있다
 - DAO 메소드에서 사용하는 SQL 문장을 UserDao 코드가 아니라 외부 리소스에 담고 이를 읽어와 사용하게 할 수 있다.
 
 
  3-7 정리
 ---------------------------------------------------------------------------
   * JDBC와 같은 예외가 발생할 가능성이 있으며 공유 리소스의 반환이 필요한 코드는 반드시 try/catch/finally 블록으로 관리해야 한다.
   * 일정한 작업 흐름이 반복되면서 그중 일부 기능만 바뀌는 코드가 존재한다면 전략 패턴을 적용한다. 바뀌지 않는 부분은 컨텍스트로, 바뀌는 부분은 전략으로 만들고 인터페이스를 통해 유연하게 전략을 변경할 수 있도록 구성한다 (전략패턴)
   * 같은 어플리케이션 안에서 여러 가지 종류의 전략을 다이나맥하게 구성하고 사용해야 한다면 컨텍스트를 이용하는 클라이언트 메소드에서 직접 전략을 정의하고 제공하게 만든다.
   * 클라이언트 메소드 안에 익명 내부 클래스를 사용해서 전략 오브젝트를 구현하면 코드도 간결해지고 메소드의 정보를 전달하지 않고 직접 사용할 수 있다.
   * 컨텍스트가 하나 이상 클라이언트 오브젝트에서 사용된다면 클래스를 분리해서 공유하도록 만든다.
   * 컨텍스트는 별도의 빈으로 등록해서 DI 받거나 클라이언트 클래스에서 직접 생성해서 사용한다. 클래스 내부에서 컨텍스트를 사용할 때 컨텍스트가 의존하는 외부의 오브젝트가 있다면 코드를 이용해서 직접 DI 받을수 있다.
   * 단일 전략 메소드를 갖는 전략패턴이면서 익명 내부 클래스를 사용해서 매번 전략을 새로 만들어 사용하고, 컨텍스트 호출과 동시에 전략 DI를 수행하는 방식을 템플릿/콜백 패턴이라고 한다.
   * 콜백 코드에도 일정한 패턴이 반복된다면 콜백을 템플릿에 넣고 재활용하는것이 편리하다.
   * 템플릿과 콜백의 타입이 다양하게 바뀔 수 있다면 제네릭스를 이용한다.
   * 스프링은 JDBC 코드 작성을 위해 jdbcTempalte을 기반으로 하는 다양한 템플릿과 콜백을 제공한다.
   * 템플릿은 한 번에 하나 이상의 콜백을 사용할 수 도 있고, 하나의 콜백을 여러번 호출할 수도 있다.
   * 템플릿/콜백을 설게할 때는 템플릿과 콜백 사이에 주고받는 정보에 관심을 두자.
    