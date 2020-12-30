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

가장 편하고 좋은 테스트 방법은 가능한 작은 단위로 쪼개서 테스트하는 것이다.
 - 테스트가 실패했을 때 원인을 쉽게 찾을 수 있기 쉽다
 - 테스트의 의도내 내용이 분명해진다
 - 만들기 쉬워진다

=> 하지만, 테스트 대상이 다른 오브젝트와 환경에 의존하고 있다면 작은 단위의 테스트가 주는 장점을 얻기 힘들어진다.


복잡한 의존관계 속의 테스트

UserServiceTest가 테스트하고자 하는 대상인 UserService는 사용자 정보를 관리하는 비지니스 로직의 구현 코드이다.
하지만 UserService는 UserDao, TransactionManager, MailSender라는 세가지 의존 관계를 갖고 있다.
따라서 세가지 의존관계를 갖는 오브젝트들이 테스트가 진행 되는 동안에 같이 실행된다.
게다가 세가지 의존 오브젝트들은 자신의 코드만 실행하는것이 아닌 또 다른 의존관계를 가지고, 이중 어떤것이라도 셋업되어 잇지 않거나 코드에 문제가 있다면 UserService에 대한 테스트가 실패한다.


테스트 대상 오브젝트 고립시키기

앞서 살펴본 이유 때문에 테스트 대상의 환경, 외부 서버, 다른 클래스 등에 영향을 받지 않도록 고립시킬 필요가 있다. => 테스트 대역을 적용하자

(그림 6-6)
의존 오브젝트나 외부 서비스에 의존하지 않는 고립된 테스트 방식으로 만든 UserServiceImpl는 아무리 기능을 수행해도 그 결과가 DB를 통해 남지 않으니, 기존의 방법으로는 작업결과를 검증하기 힘들다.
이럴때는 테스트 대상인 UserServiceImpl와 그 협력 오브젝트인 UserDao에게 어떤 요청을 했는지 확인하는 작업이 필요하다.
즉, UserDao와 같은 역할을 하면서 UserServiceImpl와 주고받은 정보를 저장해뒀다가 테스트 검증에 사용할 수 있게 하는 목 오브젝트를 만들 필요가 있다.


고립된 단위 테스트 활용

기존 테스트 코드의 구성은 아래와 같다
1. 테스트 실행중에 UserDao를 통해 가져올 테스트용 데이터를 DB에 넣는다.
2. 메일 발송여부를 확인하기 위해 MailSender 목 오브젝트를 DI 해준다.
3. 실제 테스트 대상인 userService의 메소드를 실행한다.
4. 결과가 DB에 반영되었는지 확인하기 위하여 UserDao를 통해 DB에서 데이터를 가져와 결과를 확인한다.
5. 목 오브젝트를 통해 UserService에 의한 메일 발송이 있었는지 확인한다.


UserDao 목 오브젝트

이제 실제 UserDao와 DB까지 직접 의존하고 있는 1, 4번 과정을 목 오브젝트를 만들어서 적용해보자
목 오브젝트는 기본적으로 스텁고 같은 방식으로 테스트 대상을 통해 사용될 때 필요한 기능을 지원해줘야 한다.
upgradeLevels 메소드가 실행되는 중에 UserDao와 어떤 정보를 주고받는지 입출력 내역을 먼저 확인할 필요가 있다.
아래 코드에서 보듯 UserServiceImpl를 보면 upgradeLevel() 메소드와 그 사용 메소드에서 UserDao를 사용하는 경우는 2가지 이다.


```java
public void upgradeLevels() {
  List<User> users = userDao.getAll(); // 업그레이드 후보 사용자 목록을 가져온다.
  for(User user : users) {
    if (canUpgradeLevel(user)) {
      upgradeLevel(user);
    }
  }
}

protected void upgradeLevel(User user) {
  user.upgradeLevel();
  userDao.update(user); // 수정된 사용자 정보를 DB에 반영한다.
  sendUpgradeEMail(user);
}
```

1. userDao.getAll() 업그레이드 후보 사용자 목록을 가져온다.은 이 메소드 기능을 지원하기 위해서 테스트용 UserDao에는 DB에서 읽어온 것 처럼 미리 준비된 사용자 목록을 제공해줘야한다.
2. userDao.update(user) 의 호풀은 리턴값이 없기 때문에 테스트용 UserDao가 특별히 미리 준비해둘 것은 없다 하지만 upgradeLevel의 핵심 로직인 '전체 사용자 중 업그레이드 대상자는 레벨을 변경해준다' 에서
'변경'에 해당하는 부분을 검증할 수 있는 중요한 기능이기도 하다.
   
=> getAll()에 대해서는 스텁으로서, update()에 대해서는 목 오브젝트로서 동작하는 UserDao 타입의 테스트 대역이 필요하다. (MockUseDao)

MockUserDao의 코드는 아래와 같다.

```java
public class MockUserDao implements UserDao {
  private List<User> users; // 레벨 업그레이드 후보 User 목록
  private List<User> updated = new ArrayList(); // 업그레이드 대상 User 저장해둘 리스트

  public MockUserDao(List<User> users) {
    this.users = users;
  }

  public List<User> getUpdated() {
    return this.updated;
  }

  public List<User> getAll() { // 스텁 기능 제공
    return this.users;
  }

  public void update(User user) { // 목 오브젝트 기능 제공
    updated.add(user);
  }
  
  public void add(User user) {throw new UnsupportedOperationException();}
  public User get(String id) {throw new UnsupportedOperationException();}
  public void deleteAll() {throw new UnsupportedOperationException();}
  public int getCount() {throw new UnsupportedOperationException();}
}

```

MockUserDao는 UserDao 구현 클래스를 대신해야 하니 당영ㄴ히 UserDao 인터페이스를 구현해야한다. 사용하지 않을 메소드는 호출되면 예외를 던지도록 만든다.
users는 생성자를 통해 전달받은 사용자 목록을 저장해뒀다가 getAll() 메소드가 호출되면 DB에서 가져온 것 처럼 반환해준다.
updated는 update() 메소드를 실행하면서 넘겨준 업그레이드 대상 User 오브젝트를 저장해뒀다가 검증을 위해 돌려주기 위한 것이다.


upgradeLevels() 테스트가 MockUserDao를 사용하도록 수정한 코드는 아래와 같다.

```java
@Test
public void upgradeLevels() throws Exception {
  UserServiceImpl userServiceImpl = new UserServiceImpl(); // 고립된 테스트에서는 테스트 대상 오브젝트를 직접 생성하면 된다.

  MockUserDao mockUserDao = new MockUserDao(this.users); // 목 오브젝트로 만 UserDao를 직접 DI 해준다.
  userServiceImpl.setUserDao(mockUserDao);

  userServiceImpl.upgradeLevels();

  List<User> updated = mockUserDao.getUpdated(); // MockUserDao로 부터 업데이트 결과를 가져오고 확인한다.
  assertEquals(updated.size(), 2);
  checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER);
  checkUserAndLevel(updated.get(1), "madnite1", Level.GOLD);

}

private void checkUserAndLevel(User updated, String expectedId, Level expectedLevel) {
  assertEquals(updated.getId(), expectedId);
  assertEquals(updated.getLevel(), expectedLevel);
}
```

테스트 대역 오브젝트를 이용해 완전히 고립된 테스트로 만들기 전의 테스트의 대상은 스프링 컨테이너에서 @Autowired를 통해 가져온 UserService 타입의 빈이었다.
이제는 완전히 고립돼서 테스트만을 위하 독립적으로 동작하는 테스트 대상을 사용할 것이기 때문에 스프링 컨테이너에서 빈을 가져올 필요가 없다.


테스트 수행 성능의 향상

워낙 간단한 테스트라 전체 수행시간의 차이를 못 느끼겠지만 upgradeLevels()의 테스트 수행시간은 이전보다 분명히 빨라졌다.
그 이유는 UserServiceImpl와 테스트를 도와주는 두 목 오브젝트 외에는 사용자 관리 로직을 검증하는데 직접적으로 필요하지 않은 의존 오브젝트와 서비스를 모두 제거한 덕분이다.
고립된 테스트를 하면 테스트가 다른 의존 대상에 영향을 받을 경우를 대비해 복잡하게 준비할 필요가 없을 뿐 아니라, 테스트 수행성능도 크게 향상된다.


단위 테스트와 통합 테스트

단위 테스트의 단위는 정하기 나름이다. 사용자 관리 기능 전체를 하나의 단위로 볼 수도 있고, 하나의 클래스나 하나의 메소드를 단위로 볼 수도 있다.
중요한 것은 하나의 단위에 초첨을 맞춘 테스트라는 것이다.

이 책에서는 '단위 테스트' 와 '통합 테스트'를 아래와 같이 정의하고 있다.
 - 단위 테스트: 테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시킨 테스트
 - 통합 테스트: 2개 이상의 성격이나 계층이 다른 오브젝트가 연동하도록 만들어진 테스트 또는 외부 리소스가 참여하는 테스트

그렇다면 단위 테스트와 통합 테스트 중에서 어떤 방법을 쓸지 어떻게 결정할 것인가?
 - 항상 단위 테스트를 먼저 고려한다.
 - 하나의 클래스 또는 성격과 목적이 같은 긴밀한 클래스 몇개를 모아서 외부와의 의존관계를 모두 차단하고 필요에 따라 스텁이나 목 오브젝트 등의 테스트 대역을 이용하도록 테스트를 만든다.
 - 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다.
 - 여러 개의 단위가 의존관계를 가지고 동작할 때를 위한 통합 테스트는 필요하다. (다만 단위 테스트를 충분히 거쳤다면 통합 테스트의 부담은 상대적으로 줄어든다.)
 - 단위 테스트를 만들기가 너무복잡하다고 판단되는 코드는 처음부터 통합 테스트를 고려해본다.트 (이때도 단위 테스트가 가능한 부분을 미리 단위 테스트로 만들어두면 통합 테스트의 부담이 줄어든다.)
 - 스프링 테스트 컨텍스트 프레임워크를 이용하는 테스트는 통합 테스트이다.

테스트는 코드가 작성되고 빠르게 진행되는 편이 좋다. 코드를 만들고 나서 오랜 시간이 지난 뒤에 작성하는 테스트는 테스트 대상 코드에 대한 이해가 떨어지기 때문에 불완전해지기 쉽고 작성하기도 번거롭다.
코드를 작성하면서 테스트는 어떻게 만들 수 있을까를 생각해보는 것은 좋은 습관이다. 테스트하기 편하게 만들어진 코드는 깔끔하고 좋은 코드가 될 가능성이 높다.

스프링이 지지하고 권장하는 깔끔하고 유연한 코드를 만들다보면 테스트도 그만큼 만들기 쉬워지고, 테스트는 다시 코드의 품질을 높여주고, 리펙토링과 개선에 대한 용기를 주기도 할것이다.
반대로 좋은 코드를 만들려는 노력을 게을리 한다면 테스트를 작성하기 불편해지고 테스트를 잘 만들지 않게 될 가능성이 높아진다. 테스트가 없으니 과감하게 리펙토링 할 엄두를 내지 못할 것이고 코드의 품질은 점차 떨어질 것이다.


목 프레임워크

단위 테스트를 만들기 위해서는 스텁이나 목 오브젝트의 사용이 필수적이다. 의존관계가 없는 단순한 클래스나 세부 로직을 검증하기 위해 메소드 단위로 테스트할 때가 아니라면 대부분 의존 오브젝트를 필요로 하는 코드를 테스트 하게 되기 때문이다.
단위 테스트가 많은 장점이 있고 가장 우선시해야 할 테스트 방법인 건 사실이지만 작성이 번거롭다는 점이 문제다.
특히 목 오브젝트를 만드는 일이 가장 큰 부담이다. 
다행히도, 이런 번거로운 목 오브젝트를 편리하게 작성할 수 있도록 도와주는 목 오브젝트 지원 프레임워크가 있다.


Mockito 프레임워크

Mocktio와 같은 목 프레임워크의 특징은 목 클래스를 일일히 준비해둘 필요가 없다는 점이다. 간단한 메소드 호출만으로 동적으로 특정 인터페이스를 구현한 테스트용 목 오브젝트를 만들 수 있다.
UserDao 인터페이스를 구현한 테스트용 목 오브젝트는 다음과 같이 Mockito 스태틱 메소드를 한번 호출해 주면 만들어진다.
```java
    UserDao mockUserDao = mock(UserDao.class);
```
이렇게 만들어진 목 오브젝트는 아무런 기능이 없다. 스텁기능을 추가해보자.

```java
    when(mockUserDao.getAll()).thenReturn(this.user);
```
이렇게 선언해두면 mockUserDao.getAll()이 호출 되었을 때 users를 리턴하라는 선언이다.

다음은 update() 호출이 있었는지 검증하는 부분이다. Mockito를 통해 만들어진 목 오브젝트는 메소드의 호출과 관련된 모든 내용을 자동으로 저장해두고 이를 간단한 메소드로 검증할 수 있게 해준다.
테스트를 하는 동안 mockUserDao의 update() 메소드가 2번 호출 됐는지 확인하고 싶다면 아래과 같이 검증 코드를 추가한다.
```java
    verify(mockUserDao, times(2)).update(any(User.class));
```
이렇게 선언해두면 User 타입의 오브젝트를 받으며 update()메소드가 2번 호출되었는지 확인하라는 것이다.

Mockito 목 오브젝트는 다음 4단계를 거쳐 사용하면된다. (2,4는 필요할 경우에만 사용할 수 있다)
1. 인터페이스를 통해 목 오브젝트를 만든다.
2. 목 오브젝트가 리턴할 값이 있으면 이를 지정해둔다. 메소드가 호출되면 예외를 강제로 던지게 만들수도 있다.
3. 테스트 대상 오브젝트에 DI해서 목 오브젝트가 테스트 중에 사용되도록 만든다.
4. 테스트 대상 오브젝트를 사용한 후에 목 오브젝트의 특정 메소드가 호출되었는지, 어떤 값을 가지고 몇 번 호출되었는지 검증로직을 추가해둔다.

아래는 Mocktio를 이용해 만든 테스트 코드이다.
```java
@Test
public void upgradeLevels() throws Exception {
  UserServiceImpl userServiceImpl = new UserServiceImpl();

  UserDao mockUserDao = mock(UserDao.class);
  when(mockUserDao.getAll()).thenReturn(this.user);
  userServiceImpl.setUserDao(mockUserDao);
  
  MailSender mockMailSender = mock(MailSender.class); // 리턴 값이 없는 메소드를 가진 목 오브젝트는 더욱 간단하게 만들 수 있다.
  userServiceImpl.setMailSender(mockMailSender);

  userServiceImpl.upgradeLevels();

  verify(mockUserDao, times(2)).update(any(User.class));
  
  verify(mockUserDao).update(users.get(1));
  assertThat(users.get(1).getLevel(), is(Level.SILVER));
  
  verify(mockUserDao).update(users.get(3));
  assertThat(users.get(3).getLevel(), is(Level.GOLD));
  
  ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
  verify(mockMailSender, times(2)).send(mailMessageArg.capture());
  List<SimpleMailMessage> mailMessage = mailMessageArg.getAllValues();
  
  assertThat(mailMessage.get(0).getTo()[0], is(user.get(1).getEmail()));
  assertThat(mailMessage.get(1).getTo()[0], is(user.get(3).getEmail()));
}

private void checkUserAndLevel(User updated, String expectedId, Level expectedLevel) {
  assertEquals(updated.getId(), expectedId);
  assertEquals(updated.getLevel(), expectedLevel);
}
```

Mockito는 지금까지 나온 목 오브젝트 방식을 지원하는 프레임워크 중에서 가장 사용하기 편리한 기능을 갖고 있다. 
처음엔 조금 어렵게 느껴질지 모르겟지만 목 프레임워크만의 사용방법에 익숙해지면 빠른 속도로 단위 테스트를 만들 수 있다.


6-3 다이내맥 프록시와 팩토리 빈
----------------------------------------------------------------------------

