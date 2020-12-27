# Chapter 06 AOP

AOP는 스프링의 기반기술 중 하나이다. AOP를 제대로 이용하려면 AOP의 등장배경과 AOP를 적용해 얻을 수 있는 장점이 무엇인지에 대한 이해가 필요하다.
AOP의 대포적인 사용예시인 선언전 트랜잭션 기능을 적용해보면서 AOP의 등장배경과 장점을 알아보자.

6-1 트랜잭션 코드의 분리
----------------------------------------------------------------------------
UserService 코드에 트랜잭션 경계설정을 위한 코드가 있어서 비지니스 로직와 완전히 분리가 되지 않은 것 처럼 보인다. 
하지만 논리적으로 따져보앗을 때 트랜잭션의 경계는 분명 비지니스로직의 전후에 설정되어야 하는 것이 분명하니 UserService의 메소드에 둘 수 밖에 없다.

메소드 분리

```java
public void upgradeLevels() throws Exception {
  TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try{
    
      
      List<User> users = userDao.getAll();
      for(User user : users) {
      if (canUpgradeLevel(user)) {
        upgradeLevel(user);
      }
      
      
      this.transactionManager.commit(status);
    } catch(Exception e) {
      this.transactionManager.rollback(status);
      throw e;
    }
    
  }
```
- 자세히 살펴보면 두 가지 종류의 코드가 구분되어있음
- 트랙잭션 경계설정의 코드와 비지니스 로직 코드 사이 서로 주고받는 정보가 없다

=> 따라서 이 두가지 코드는 성격이 다를 뿐 아니라 서로 주고받는 것도 없는 완벽하게 독립적인 코드다.

```java
public void upgradeLevels() throws Exception {
  TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try{
      upgradeLevelsInternal();
      this.transactionManager.commit(status);
    } catch(Exception e) {
      this.transactionManager.rollback(status);
      throw e;
    }
  }

  private void upgradeLevelsInternal() {
    List<User> users = userDao.getAll();
    for(User user : users) {
      if (canUpgradeLevel(user)) {
        upgradeLevel(user);
      }
    }
  }
```

=> 비지니스로직 코드를 extract method 하여 분리하였다.


DI를 이용한 클래스의 분리

트랜잭션 코드를 UserService 밖으로 빼버리면 UserService 클래스를 직접 사용하는 클라이언트 코드에서는 트랜잭션 기능이 빠진 UserService를 사용하게 될 것

=> 직접 사용하는것이 문제가 된다면 간접적으로 사용하게 한다 즉, DI를 적용한다

=> UserService를 인터페이스로 만들고 기존 코드는 UserService 인터페이스의 구현 클래스로 만들어넣도록 한다. 그러면 클라이언트와 결합이 약해지고 직접 구현 클래스에 의존하고 잇지 않기 때문에 유연한 확장이 가능해진다.

```java
public interface UserService {
  void add(User user);
  void upgradeLevels();
}
```
그런데 보통 이렇게 인터페이스를 이용해 구현 클래스를 클라이언트에 노출하지 않고 런타임 시에 DI를 통해 적용하는 방법을 쓰는 이유는 일반적으로 구현 클래스를 바꿔가면서 사용하기 위함이다. 하지만 꼭 그렇게 사용해야한다는 제약은 없고 한번에 두개의 UserService 인터페이스 구현 클래스를 동시에 이용해도 된다.

=> UserService를 구현한 또 다른 구현클래스를 만든다. 이 클래스는 사용자 관리 로직을 담고 있는 구현클래스인 UserServiceImpl를 대체하기 위한 것이 아닌 트랜잭션 경계설정을 위한 구현 클래스이다.

UserService 인터페이스의 구현 클래스인 UserSerivceImpl은 기존 UserService 클래스의 내용을 대부분 그대로 유지한다. 단 트랜잭션과 관련된 코드는 독립시키기로 했으니 모두 제거한다.

```java
public class UserSerivceImpl implemets UserService {
    UserDao userDao;
    MailSender mailSender;
    
    public void upgradeLevels() {
      List<User> users = userDao.getAll();
      for(User user : users) {
        if (canUpgradeLevel(user)) {
          upgradeLevel(user);
        }
      }
    }    
}
```

이제 비지니스 트랜잭션 처리를 담은 UserServiceTx를 만들어보자. UserService를 구현하게 만들고 같은 인터페이스를 구현한 다른 오브젝트에게 작업을 위임하게 한다.

```java
public class UserServiceTx implements UserService {
  UserService userService;

  public void setUserService(UserService userService){
    this.userService = userService;
  }

  public void add(User user){
    userService.add(user);
  }

  public void upgradeLevels() {
    userService.upgradeLevels();
  }
}
```

UserServiceTx는 UserService 인터페이스를 구현했으니 클라이언트에 대해 UserService 타입 오브젝트의 하나로서 행세할 수 있다.
UserServiceTx는 사용자 관리라는 비지니스로직을 전혀 갖지 않고 고스란히 다른 UserService 구현 오브젝트에 기능을 위임한다.

이렇게 준비된 UserServiceTX에 트랜잭션의 경계설정이라는 부가적인 작업을 부여해준다.

```java
public class UserServiceTx implements UserService {
  UserService userService;
  PlatformTransactionManager transactionManager;

  public void setTransactionManager(PlatformTransactionManager transactionManager) {
    this.transactionManager = transactionManager;
  }

  public void setUserService(UserService userService){
    this.userService = userService;
  }

  public void add(User user){
    userService.add(user);
  }


  public void upgradeLevels() {
    TransactionStatus status = this.transactionManager.getTransaction((new DefaultTransactionDefinition()));
    try {
      userService.upgradeLevels();

      this.transactionManager.commit(status);
    } catch (RuntimeException e) {
      this.transactionManager.rollback(status);
      throw e;
    }
  }
}
```

트랜잭션 적용을 위한 DI 설정

스프링의 DI 설정에 의해 만들어질 빈 오브젝트와 의존관계는 아래와 같이 구성되어야 한다.

Client(UserServiceTest) -> UserServiceTx -> UserServiceImpl

```xml
 <bean id="userService" class="com.example.demo.user.service.UserServiceTx">
    <property name="transactionManager" ref="transactionManager"/>
    <property name="userService" ref="UserServiceImpl"/>
</bean>

<bean id="UserServiceImpl" class="com.example.demo.user.service.UserServiceImpl">
    <property name="userDao" ref="userDao"/>
</bean>
```

이제 클라이언트는 UserServiceTx 빈을 호출해서 사용하도록 만들어야 한다. 따라서 userService라는 대표적인 빈 아이디는 UserServiceTx 클래스로 정의된 빈에게 부여해준다.
userService 빈은 UserServiceImpl 클래스로 정의되는 아이디가 userServiceImpl인 빈을 DI 하게 만든다.

트랜잭션 경계설정 코드의 분리와 DI를 통한 연결을 통해 얻은것?
- 비지니스 로직을 담당하고 있는 UserServiceImpl 코드를 작성할 때 트랜잭션과 같은 기술적인 내용에는 신경쓰지 않아도 됨
- 비지니스 로직에 대한 테스트를 더 손쉽게 만들 수 있음

6-2 고립된 단위 테스트
----------------------------------------------------------------------------

