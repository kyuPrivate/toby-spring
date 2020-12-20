# Chapter 05 서비스 추상화

자바에는 표준 스펙, 상용 제품, 오픈소스를 통틀어서 사용 바업ㅂ과 형식은 다르지만 기능과 목적이 유사한 기술이 존재한다.
환경과 상황에 따라서 기술이 바뀌고, 그에 따라 다른 API를 사용하고 다른 스타일의 접근 방법을 따라야 한다는 건 매우 피곤한 일이다.
5장에서는 스프링이 어떻게 성격이 비슷한 여러 종류의 기술을 추상화하고 이를 일관된 방법으로 사용할 수 있도록 지원하는지를 살펴볼 것이다.

5-1 사용자 레벨 관리 기능 추가
----------------------------------------------------------------------------
구현해야할 비지니스 로직

- 사용자의 활동내역을 참고해 레벨을 조정
- 인터넷 서비스의 사용자 관리 기능에서 구현해야할 비즈니스 로직
- 사용자 레벨은 BASIC, SILVER, GOLD 중 하나
- 사용자가 처음 가입하면 BASIC 레벨이며, 이후 활동에 따라 한단계씩 업그레이드
- 가입 후 50회 이상 로그인을 하면 BASIC에서 SILVER 레벨이 된다.
- SIVER 레벨이면서 30번 이상 추천을 받으면 GOLD

필드 추가

```java
public class User {
  private static final int BASIC = 1;
  private static final int SILVER = 2;
  private static final int GOLD = 3;

  int level;

  public void setLevel(int level) {
    this.level = level;
  }
```
위와 같이 상수 값을 정해놓고 int 타입으로 레벨을 사용한다고 하면 발생할 수 있는 문제점
- level 필드의 타입이 int이기 떄문에 user.setLevel(otherUser.getSum()); 러럼 다른 종류의 정보를 넣는 실수를 해도 컴파일러가 잡아줄 수 없다.
- 우연히 다른 종류의 정보(getSum)가 1,2,3 과 같은 유효한 레벨값을 돌려주면 문제없이 동작하는 것 처럼 보이겠지만 사실은 엉뚱한 레벨이 들어가는 심각한 버그가 발생할 수 있다.

=> 숫자 타입을 직접 사용하는 것 보다 enum을 이용하는게 안전하고 편리하다.

```java
public enum Level {
  BASIC(1), SILVER(2), GOLD(3);

  private final int value;

  Level(int value) {
    this.value = value;
  }

  public int intValue() {
    return value;
  }

  public static Level valueOf(int value) {
    switch(value) {
      case 1: return BASIC;
      case 2: return SILVER;
      case 3: return GOLD;
      default: throw new AssertionError("Unknown value: " + value);
    }
  }
}
```
이렇게 만들어진 enum은 내부에는 DB에 저장할 int 타입의 값을 가지고 있지만 겉으로는 Level 타입의 오브젝트이기 때문에 오용하지 않고 안전하게 사용할 수 있다. (user.setLevel(1) 과 같은 코드는 컴파일러에서 타입 불일치 에러를 발생시킬 것)

User 필드 추가

```java
public class User {
  Level level;
  int login;
  int recommend;
  String id;
  String name;
  String password;

  public User(String id, String name, String password) {
    this.id = id;
    this.name = name;
    this.password = password;
  }

  public User(String id, String name, String password, Level level, int login, int recommend) {
    this.id = id;
    this.name = name;
    this.password = password;
    this.level = level;
    this.login = login;
    this.recommend = recommend;
  }

  public User() {}

   // getter setter 생략
}
```

UserDaoTest 테스트 수정

기존 코드에 새로운 기능을 추가하려면 테스트를 먼저 만드는 것이 안전하다.
먼저 새로운 픽스처로 만든 user1, user2, user3에 새로만든 필드의 값을 넣는다.

```java
public class UserDaoTest {
...
  @Before
  public void setUp(){
    this.user1 = new User("gyumee", "박성철", "springno1", Level.BASIC, 1, 0);
    this.user2 = new User("leegw700", "이길원", "springno2", Level.SILVER, 55, 10);
    this.user3 = new User("bumjin", "박범진", "springno3", Level.GOLD, 100, 40);
  }
```

UserDaoTest 에서 두 개의 User 오브젝트 필드 값이 모두 같은지 비교하는 checkSameUser() 메소드도 수정한다.

```java
private void checkSameUser(User user1, User user2){
  assertEquals(user1.getId(), user2.getId());
  assertEquals(user1.getName(), user2.getName());
  assertEquals(user1.getPassword(), user2.getPassword());
  assertEquals(user1.getLevel(), user2.getLevel());
  assertEquals(user1.getLogin(), user2.getLogin());
  assertEquals(user1.getRecommend(), user2.getRecommend());
}
```

UserDaoJdbc 수정

리스트 5-9 추가된 필드를 위한 UserDaoJdbc

```java
public class UserDaoJdbc implements UserDao {
...
    private RowMapper<User> userMapper = 
        new RowMapper<User>() {
            public User mapRow(ResultSet rs, int rowNum) throws SQLException {
                User user = new User();
                user.setId(rs.getString("id"));
                user.setName(rs.getString("name"));
                user.setPassword(rs.getString("password"));
                user.setLevel(Level.valueOf(rs.getString("level"));
                user.setLogin(rs.getInt("login"));
                user.setRecommend(rs.getInt("recommend"));
            }
        };
        
     public void add(User user) {
        this.jdbcTemplate.update(
            "insert into users(id, name, password, level, login, recommend)" +
            "values(?,?,?,?,?,?)", user.getId(), user.getName(), user.getPassword(), user.getLevel().intValue(), user.getLogin(), user.getRecommend());
        );
     }
}
```
여기서 눈여겨 볼 것은 Level 타입의 level 필드를 사용하는 부분이다. Level은 Enum이고, Enum은 오브젝트로 DB에 저장될 수 있는 SQL 타입이 아니다. 따라서 DB에 저장가능한 형태로 변환해준다.
반대로 조회했을 경우에는 int로 level정보를 가져와서 Level의 스태틱 메소드인 valueOf()를 사용하여 int값을 Level 오브젝트로 만들어서 setLevel 메소드에 넣어준다.


수정 기능 테스트 추가

만들어야 할 기능을 점검해 볼 겸 테스트를 먼저 작성한다.

```java
@TEST
public void update(User user) {
    dao.deleteAll();
   
    dao.add(user1);
    
    user1.setName("오민규");
    user1.setPassword("qwer1234");
    user1.setLevel(Level.GOLD);
    user1.setLogin(100);
    user1.setRecommend(1234);
    dao.update(user1);
    
    User user1updated = dao.get(user1.getId());
    checkSameUser(user1, user1updated);
}
```

UserDao 인터페이스에 update 메소드를 추가하고 UserDaoJdbc에도 update 메소드 구현을 추가한다.

```java
public interface UserDao {
  ...
  void update(User user1);
}

public class UserDaoJdbc implements UserDao {
...
    public void update(User user) {
      this.jdbcTemplate.update("update users set name = ?, password = ?, level = ?, login = ?, recommend = ? where id = ?", user.getName(), user.getPassword(), user.getLevel().intValue(), user.getLogin(), user.getRecommend(), user.getId());
    }
}
```

문제점? update 문장에서 where절을 빼먹는 경우에 대한 검증이 안됨
1. JdbcTemplate의 update가 반환하는 리턴값을 확인한다. 리턴값은 update가 영향받은 row의 수를 돌려준다
2. 테스트를 보강해서 원하는 사용자 외의 정보는 변경되지 않음을 확인한다. (아래와 같다)

```java
@Test
public void update() {
  dao.deleteAll();

  dao.add(user1);
  dao.add(user2);

  user1.setName("오민규");
  user1.setPassword("springno6");
  user1.setLevel(Level.GOLD);
  user1.setLogin(1000);
  user1.setRecommend(999);
  dao.update(user1);

  User user1update = dao.get(user1.getId());
  checkSameUser(user1, user1update);
  User user2same = dao.get(user2.getId());
  checkSameUser(user2, user2same);
}
```

UserService.upgradeLevel()

사용자의 레벨 관리 로직을 구현하자. 이 메소드를 구현할 곳으로 UserDaoJdbc는 적당하지 않다. DAO는 데이터를 어떻게 가져오고 조작할지를 다루는 곳이지 비지니스 로직을 두는 곳이 아니다.
사용자 관리 비지니스 로직을 담을 클래스를 비지니스 로직을 제공한다는 의미에서 UserService로 한다.
인터페이스 타입으로 UserDao를 DI받아 사용하게 한다. (UserService는 UserDao의 구현클래스가 바뀌어도 영향받지 않아야 한다. 데이터 액세스 로직이 바뀌더라도 비지니스 로직이 바뀌는건 아니기 때문이다. 따라서 DAO 인터페이스를 사용하고 DI 적용해야 한다.)

```java
public class UserService {
  UserDao userDao;

  public void setUserDao(UserDao userDao) {
    this.userDao = userDao;
  }
}
```

```xml
<bean id="userService" class="springbook.user.service.UserService">
  <property name="userDao" ref="userDao"/>
</bean>

<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
  <property name="dataSource" ref="dataSource"/>
</bean>
```


UserServiceTest 클래스

```java
@ContextConfiguration(locations="/test-applicationContext.xml")
public class UserServiceTest {
  @Autowired
  UserService userService;
  
  @Test
  public void bean() {
    assertThat(this.useService, is(notNullValue()));
  }
}
```

upgradeLevel 메소드

```java
public class UserService {
  UserDao userDao;

  public void setUserDao(UserDao userDao) {
    this.userDao = userDao;
  }
  
  public void upgradeLevels() {
  List<User> users = userDao.getAll();
  for(User user : users){
    Boolean changed = null;
    if(user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
      user.setLevel(Level.SILVER);
      changed = true;
    } else if (user.getLevel() == Level.SILVER && user.getRecommend() >= 30) {
      user.setLevel(Level.GOLD);
      changed = true;
    } else if (user.getLevel() == Level.GOLD) {
      changed = false;
    } else {
      changed = false;
    }

    if(changed) {
      userDao.update(user);
    }
  }
}
}
```

upgradeLevel 테스트

BASIC, SILVER, GOLD에서 변경되지 않는 3케이스 + BASIC -> SILVER, SILVER -> GOLD 변경되는 2케이스
테스트 할때 값은 경계값으로 하는게 좋다.

```java
@ContextConfiguration(locations="/test-applicationContext.xml")
public class UserServiceTest {
  @Autowired
  UserService userService;
 
  List<User> users;
  
  @Before
  public void setUp() {
        users = Arrays.asList(
          new User("bumjin", "박범진", "p1", Level.BASIC, 49, 0),
          new User("joytouch", "강명성", "p2", Level.BASIC, 50, 10),
          new User("erwins", "신승한", "p3", Level.SILVER, 60, 29),
          new User("madnite1", "이상호", "p4", Level.SILVER, 60, 30),
          new User("green", "오민규", "p5", Level.GOLD, 100, 100)
      );
  }
  
    @Test
    public void upgradeLevels() {
      userDao.deleteAll();
      for(User user: users) {
        userDao.add(user);
      }
    
      userService.upgradeLevels();
    
      checkLevel(users.get(0), Level.BASIC);
      checkLevel(users.get(1), Level.SILVER);
      checkLevel(users.get(2), Level.SILVER);
      checkLevel(users.get(3), Level.GOLD);
      checkLevel(users.get(4), Level.GOLD);
    }
    
    private void checkLevel(User user, Level expectedLevel) {
      User userUpdate = userDao.get(user.getId());
      assertEquals(userUpdate.getLevel(), expectedLevel);
    }
}
```

UserService.add()

처음 가입하는 사용자는 기본적으로 BASIC 레벨이어야 한다는 부분을 검증하자. UserDaoJdbc의 add메소드에 비지니스 로직을 담는것은 부적절하다.
=> 사용자 관리에 대한 비지니스 로직을 담고 있는 UserService에 이 로직을 넣어야 한다.

```java
@Test
public void add() {
  userDao.deleteAll();

  User userWithLevel = users.get(4);
  User userWithoutLevel = users.get(0);
  userWithoutLevel.setLevel(null);

  userService.add(userWithLevel);
  userService.add(userWithoutLevel);

  User userWithLevelRead = userDao.get(userWithLevel.getId());
  User userWithoutLevelRead = userDao.get(userWithoutLevel.getId());

  assertEquals(userWithLevelRead.getLevel(), userWithLevel.getLevel());
  assertEquals(userWithoutLevelRead.getLevel(), Level.BASIC);
}
```

```java
public void add(User user) {
  if (user.getLevel() == null) user.setLevel(Level.BASIC);
  userDao.add(user);
}
```

코드 개선

작성된 코드를 살펴볼 때는 다음과 같은 점을 확인해보자
- 코드에 중복된 부분은 없는가
- 코드가 무엇을 하는지 이해하기 불편하지 않은가
- 코드가 자신이 있어야할 자리에 있는가
- 어떤 변경이 발생할 수 있고, 변경에 쉽게 대응할 수 있는가

 
 upgradeLevels 메소드의 문제점
 
 - for 루프 속에 있는 if/else if/else 블록들이 읽기 불편하다
 - 레벨의 변화 단계와 업그레이드 조건, 조건이 충족되었을 때 해야할 작업이 섞여 있어서 로직을 이해하기 쉽지 않다
 - 플래그를 사용하고 있는 점이 깔끔해 보이지 않는다
 - if 조건 블록이 레벨 갯수만큼 반복되어야 한다. (레벨이 추가 된다면 Level enum과 upgradeLevel의 if 블록을 추가해줘야 한다)
 
 upgradeLevels 리펙토링
 
 ```java
public void upgradeLevels() {
  List<User> users = userDao.getAll();
  for(User user : users) {
    if (canUpgradeLevel(user)) {
      upgradeLevel(user);
    }
  }
}
```
모든 사용자 정보를 가져와 한 명씩 업그레이드가 가능한지 확인하고, 가능하면 업그레이드 한다.

```java
public boolean canUpgradeLevel(User user) {
  Level currentLevel = user.getLevel();
  switch (currentLevel) {
    case BASIC: return (user.getLogin() >= 50);
    case SILVER: return (user.getRecommend() >= 30);
    case GOLD: return false;
    default: throw new IllegalArgumentException("Unknown Level: " + currentLevel);
  }
}
```
상태에 따라서 업그레이드 조건만 비교하면 되므로, 역할과 책임이 명료해 진다.

```java
private void upgradeLevel(User user) {
  if(user.getLevel() == Level.BASIC) user.setLevel(Level.SILVER);
  else if(user.getLevel() == Level.SILVER) user.setLevel(Level.GOLD);
  userDao.update(user);
}
```
사용자 오브젝트의 레벨을 다음 단계로 변경하고 변경된 오브젝트를 DB에 업데이트 하는 작업을 수행한다.
추가로, 다음 단계가 무엇인가 하는 로직과 그때 사용자 오브젝트의 level필드를 변경해 준다는 로직이 너무 노골적으로 드러나 있고 예외 상황에 대한 처리가 없는점을 개선해보자
=> 레벨의 순서와 다음 레벨이 무엇인지 결정하는 책임을 Level enum에 맡기자.

```java
public enum Level {
  GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);

  private final int value;
  private final Level next;

  Level(int value, Level next) {
    this.value = value;
    this.next = next;
  }

  public int intValue() {
    return value;
  }

  public Level nextLevel() {
    return this.next;
  }

  public static Level valueOf(int value) {
    switch(value) {
      case 1: return BASIC;
      case 2: return SILVER;
      case 3: return GOLD;
      default: throw new AssertionError("Unknown value: " + value);
    }
  }
}
```
Level enum에 next라는 다음 단계 레벨 정보를 담을 수 있도록 필드를 추가했다.
생성자 파라미터를 추가해서 다음 단계 레벨 정보를 저장할 수 있게 한다.

사용자 정보가 바뀌는 부분을 UserService 메소드에서 User 로 옮겨보자. User의 내부 정보가 변경되는 것은 UserService보다는 User가 스스로 다루는 게 적절하다.
User도 자바 오브젝트이고 내부 정보를 다루는 기능이 있을 수 있다. UserService가 일일히 레벨업시 User의 특정 필드를 수정하기 보단 User에게 레벨업을 해야하니 정보를 변경하라고 요청하는게 낫다.

```java
public void upgradeLevel() {
  Level nextLevel = this.level.nextLevel();
  if(nextLevel == null) {
    throw new IllegalArgumentException(this.level + "은 업그레이드가 불가합니다.");
  } else {
    this.level = nextLevel;
  }
}
```
이렇게 User에서 레벨 업그레이드 작업용 메소드를 만들어두면 UserService의 upgradeLevel 메소드가 간결해진다.

```java
private void upgradeLevel(User user) {
  if(user.getLevel() == Level.BASIC) user.setLevel(Level.SILVER);
  else if(user.getLevel() == Level.SILVER) user.setLevel(Level.GOLD);
  userDao.update(user);
}
=>
private void upgradeLevel(User user) {
  user.upgradeLevel();
  userDao.update(user);
}
```

리펙토링 한 코드를 살펴보면 각 오브젝트와 메소드가 각각 자기 몫의 책임을 맡아 일을 하는 구조로 만들어졌음을 알 수 있다.
객체지향적인 코드는 다른 오브젝트의 데이터를 가져와서 작업하는 대신 데이터를 갖고 있는 다른 오브젝트에게 작업을 해달라고 요청한다.
오브젝트에게 데이터를 요구하지 말고 작업을 요처아라는 것이 객체지향 프로그래밍의 원리이다.


User 테스트

User
User에 레벨업 하는 로직을 추가했다. 이를 테스트하는 코드를 만들어보자
```java
public class UserTest {
  User user;

  @Before
  public void setUp() {
    user = new User();
  }

  @Test
  public void upgradeLevel() {
    Level[] levels = Level.values();
    for(Level level : levels){
      if(level.nextLevel() == null) continue;
      user.setLevel(level);
      user.upgradeLevel();
      assertEquals(user.getLevel(), level.nextLevel());
    }
  }

  @Test(expected=IllegalStateException.class)
  public void cannotUpgradeLevel() {
    Level[] levels = Level.values();
    for (Level level : levels) {
      if (level.nextLevel() != null) continue;
      user.setLevel(level);
      user.upgradeLevel();
    }
  }
}
```
User에 대한 테스트는 굳이 스프링 테스트 컨텍스트를 사용하지 않아도 된다. User 오브젝트는 스프링이 IoC 관리해주는 오브젝트가 아니기 때문이다.


UserServiceTest 개선

```java
기존
@ContextConfiguration(locations="/test-applicationContext.xml")
public class UserServiceTest {
  @Autowired
  UserService userService;
 
  List<User> users;
  
  @Before
  public void setUp() {
        users = Arrays.asList(
          new User("bumjin", "박범진", "p1", Level.BASIC, 49, 0),
          new User("joytouch", "강명성", "p2", Level.BASIC, 50, 10),
          new User("erwins", "신승한", "p3", Level.SILVER, 60, 29),
          new User("madnite1", "이상호", "p4", Level.SILVER, 60, 30),
          new User("green", "오민규", "p5", Level.GOLD, 100, 100)
      );
  }
  
    @Test
    public void upgradeLevels() {
      userDao.deleteAll();
      for(User user: users) {
        userDao.add(user);
      }
    
      userService.upgradeLevels();
    
      checkLevel(users.get(0), Level.BASIC);
      checkLevel(users.get(1), Level.SILVER);
      checkLevel(users.get(2), Level.SILVER);
      checkLevel(users.get(3), Level.GOLD);
      checkLevel(users.get(4), Level.GOLD);
    }
    
    private void checkLevel(User user, Level expectedLevel) {
      User userUpdate = userDao.get(user.getId());
      assertEquals(userUpdate.getLevel(), expectedLevel);
    }
}
```

```java
개선
@ContextConfiguration(locations="/test-applicationContext.xml")
public class UserServiceTest {
  @Autowired
  UserService userService;
 
  List<User> users;
  
  @Before
  public void setUp() {
        users = Arrays.asList(
          new User("bumjin", "박범진", "p1", Level.BASIC, MIN_LOGCOUNT_FOR_SILVER-1, 0),
          new User("joytouch", "강명성", "p2", Level.BASIC, MIN_LOGCOUNT_FOR_SILVER, 10),
          new User("erwins", "신승한", "p3", Level.SILVER, 60, MIN_RECOMMEND_COUNT_FOR_GOLD-1),
          new User("madnite1", "이상호", "p4", Level.SILVER, 60, MIN_RECOMMEND_COUNT_FOR_GOLD),
          new User("green", "오민규", "p5", Level.GOLD, 100, Integer.MAX_VALUE)
      );
  }
  
    @Test
    public void upgradeLevels() {
      userDao.deleteAll();
      for(User user: users) {
        userDao.add(user);
      }
    
      userService.upgradeLevels();
    
      checkLevelUpgraded(users.get(0), false);
      checkLevelUpgraded(users.get(1), true);
      checkLevelUpgraded(users.get(2), false);
      checkLevelUpgraded(users.get(3), true);
      checkLevelUpgraded(users.get(4), false);
    }
    
    private void checkLevelUpgraded(User user, boolean upgraded) {
      User userUpdate = userDao.get(user.getId());
      if(upgraded) {
        assertThat(userUpdate.getLevel(), is(user.getLevel().nextLevel());
      } else {
        assertThat(userUpdate.getLevel(), is(user.getLevel()));
      }
    }
}
```
기존 테스트에서는 checkLevel 메소드를 호출할 때 일일히 다음 단계의 레벨이 무엇인지 넣어줬다. Level이 가지고 있어야 할 다음 레벨이 무엇인가 하는 정보를 테스트에 넣을 필요가 없다.
또한 테스트와 어플리케이션 코드에 나타난 숫자의 중복도 제거했다.
숫자로만 되어있는 경우에는 비지니스 로직을 상세히 코멘트로 달아놓거나 설계문서를 참조하기 전에는 이해하기 어려웠던 부분이 이제는 무슨 의도로 값을 넣었는지 쉽게 이해할 수 있게 되었다.
또한 코드와 테스트 사이에도 중복을 제거했기 때문에 업그레이드 조건 값이 바뀌는 경우 UserSerivce의 상수값만 변경해주면 된다.

5-2 트랙잭션 서비스 추상화
----------------------------------------------------------------------------

테스트용 UserService의 대역


현재 코드에서 유저 레벨 변경작업 중 장애가 발생했을 때 이미 변경된 유저 레벨을 원래대로 돌리는지 확인해보기 위하여 테스트를 작성한다. 
=> 장애가 발생헀을 때 일어나는 현상 중 하나인 예외가 던져지는 상황을 의도적으로 만들자!

테스트를 위하여 UserService 코드를 수정하는것은 바람직하지 않기 때문에 UserService를 상속해서 테스트에 필요한 일부 메소드를 오브라이딩 하자 => upgradeLevel 메소드를 private -> protected로 접근자 수정한다.

```java
static class TestUserService extends UserService {
  private String id;

  private TestUserService(String id) { // 작업을 중단시킬 id를 설정한다.
    this.id = id;
  }

  protected void upgradeLevel(User user) {
    if (user.getId().equals(this.id)) throw new TestUserServiceException(); // 지정된 id의 User Object에서 예외를 던져 강제로 작업을 중단시킨다.
    super.upgradeLevel(user);
  }
}

static class TestUserServiceException extends RuntimeException { // 다른 예외가 발생했을 걍우와 구분하기 위하여 테스트 목적을 띈 예외를 정의하였다.
} 
```


강제 예외 발생을 통한 테스트

```java
@Test
public void upgradeAllOrNothing() {
  UserService testUserService = new TestUserService(users.get(3).getId());
  testUserService.setUserDao(this.userDao); //TestUserService는 테스트 메소드에서만 특수한 목적으로 사용되는 것이니 Bean으로 등록하지 않았기 때문에 수동으로 DI
  userDao.deleteAll();
  for(User user : users) userDao.add(user);
  try {
    testUserService.upgradeLevels();
    fail("TestUserServiceException expected"); // upgradeLevels를 완료하고 정상 종료되면 문제가 있으니 실패
  } catch (TestUserServiceException e){
  }

  checkLevelUpgraded(users.get(1), false); // 레벨 변경이 일어나기 전 레벨로 돌아왔는지 확인
}
```

네번째 사용자의 레벨변경 처리 중 두번쨰 사용자의 레벨이 변경된 채로 그대로 유지되어씩 떄문에 테스트는 실패로 끝난다.

테스트 실패의 원인 : upgradeLevels 메소드가 하나의 트렌잭션에서 동작하지 않았기 떄문

트랙잭션 경계설정

하나의 SQL 명령어는 DB가 트랜잭션을 보장해준다고 믿을 수 있다. 하지만, 앞선 예시처럼 여러개의 SQL이 사용되는 작업을 하나의 트랙잭션으로 취급해야하는 경우도 있다.

- transaction rollback : SQL이 성공적으로 DB에 수행되기 전에 문제가 발생할 경우에는 앞에서 처리한 SQL 작업도 취소함
- transaction commit : 여러 개의 SQL을 하나으 트랜잭션으로 처리하는 경우에 모든 SQL 수행작업이 성공적으로 마무리 되었다고 DB에 알려줘서 작업을 확정함


JDBC 트랜잭션의 트랜잭션 경계설정

트랜잭션을 시작하는 방법은 1가지 => connection의 autoCommit / 트랙잭션을 종료하는 방법은 2가지 => commit or rollback

```java
Connection c = dataSource.getConnection();

c.setAutoCommit(false);
try {
  PerparedStatement st1 = c.prepareStatement("update users ...");
  st1.executeUpdate();

  PerparedStatement st2 = c.prepareStatement("delete users ...");
  st2.executeUpdate();
  
  c.commit(); // 여기까지 예외가 없으면 commit 으로 트랜잭션 종료
} catch(Exception e){
  c.rollback(); // try block 에서 예외가 발생하면 수행했던 DB 작업을 rollback하고 종료
}

c.close();
```

JDBC에서 트랜잭션으르 시작하려면 autoCommit 옵션을 false로 만들어주면 된다. JDBC의 기본설정은 autoCommit = true 이므로, 하나의 SQL문이 수행되면 자동으로 커밋 되는 것이다. autoCommit = false로 설정해주면
새로운 트랙잭션이 시작되고 commit 또는 rollback 메소드가 호출될 때 까지 하나의 트랜잭션으로 묶인다.
 - transaction demarcation : setAutoCommit(false)로 트랜잭션의 시작을 선언하고 commit() 또는 rollback()으로 트랜잭션을 종료하는 작업
 - local transaction : 하나의 DB connection 안에서 만들어지는 트랙잭션
 
 
 UserService와 UserDao의 트랜잭션 문제
 
기존 UserService의 upgradeLevels에는 왜 트랙잭션이 적용되지 않았는가? => transaction demarcation 하지 않았기 때문에 기본설정으로 실행되엇음
jdbcTemplate는 하나의 템플릿 메소드 안에서 DataSource의 getConnection 메소드를 호출해서 Connection을 가져오고 작업을 마치면 Connection을 닫고 템플릿 메소드를 빠져나온다. 결국 JdbcTemplate 메소드를 사용하는 UserDao는 각 메소드마다 독립적인 트랜잭션으로 실행될 수 밖에 없다.
데이터 액세스 코드를 DAO로 만들어서 분리해놓았을 경우에는 이처럼 DAO 메소드를 호출할 때마다 하나의 새로운 트랜잭션이 만들어지는 구조가 될 수 밖에 없다.
DAO 메소드 내에서 JDBC API를 직접 이용하든, JdbcTemplate을 이용하든 DAO메소드에서 DB 커넥션을 매번 만들기 때문에 발생하는 현상이다. 결국 DAO를 사용하면 비지니스로직을 담고 잇는 UserService 내에서 진행되는 여러가지 작업을 하나의 트랜잭션으로 묶는 일이 불가능해진다.


비지니스 로직 내의 트랜잭션 경계설정

DAO 메소드 안으로 upgradeLevels 메소드의 내용을 옮기면? => 트랜잭션 문제는 해결되겠지만 비지니스 로직과 데이터 로직이 한데 묶여버린다. 성격과 책임이 다른 코드를 분리하고 느슨하게 연결해서 확장성을 확보하기 위한 노력이 의미없어진다.

UserService와 UserDao를 그대로 둔 채로 트랜잭션을 적용하려면 결국 트랜잭션의 경계설정 작업을 UserService로 가져와야 한다.
```java
public void upgradeLevels() throws Exception {
  (1) DB Connection 생성
  (2) 트랜잭션 시작
  try {
    (3) DAO 메소드 호출
    (4) 트랜잭션 커밋
  } catch(Exception e) {
    (5) 트랜잭션 롤백
    throw e;
  } finally {
    (6) DB Connection 종료
  }
}
```

UserDao의 update 메소드는 반드시 upgradeLevels 메소드에서 만든 connection을 사용해야한다. 그래야만 같은 트랙잭션에서 동작한다.
또한 upgradeLevels는 update를 직접 호출하지 않기 때문에 upgradeLevel로 connection을 재전달 해줘야 한다.

upgradeLevels (커넥션 생성) -> upgradeLevel -> update

```java
public interface UserDao {
  public void add(Connection c, User user);
  public User get(Connection c, String id);
  ...
  public void update(Connection c, User user1);
}
```

UserService 트랙잭션 경계설정의 문제점

위와 같이 수정하면 트랜잭션 문제는 해결할 수 있지만 다른 문제가 발생한다.

1. DB 커넥션을 비롯한 리소스의 깔끔한 처리를 가능하게 했던 JdbcTemplate을 더이상 활용할 수 없다. 결국 JDBC API를 직접 사용하는 초기 방식으로 돌아가야 해서 try/catch/finally 블록이 UserService내에 존재하고 JDBC 작업코드의 전형적인 문제점을 가지고 있다
2. DAO의 메소드와 비지니스 로직을 담고 있는 UserService 메소드에 Connection을 전달해주기 위한 파라미터가 주가되어야 한다. 이를 회피하기 위해 UserService 인스턴스 변수에 connection을 저장해 두면 UserService는 스프링 빈으로 등록된 객체라 싱글톤으로 되어 멀티스레드 환경에서 connection 변수를 공유하기 때문에 위험하다.
3. Connection 파라미터가 UserDao 인터페이스 메소드에 추가되면서 UserDao는 더이상 데이터 엑세스 기술에 독립적일 수 없다.
4. DAO 메소드에 Connection 파라미터를 받게 하면 테스트코드에도 영향을 미친다.


Connection 파라미터 제거

먼저, connection을 직접 파라미터로 전달하는 문제를 해결해보자. upgradeLevel 메소드가 트랜잭션 경계설정을 해야한다는 사실은 피할수 없다. 따라서 그 안에서 connection을 생성하고 트랜잭션의 시작과 종료를 관리하게 된다.
그러나 여기서 생성된 connection 오브젝트를 계속 메소드의 파라미터로 전달하다가 DAO를 호출할때 사용하는 것을 피하기 위해 transaction synchronization을 적용할 수 있다.

 - transaction synchronization : connection 오브젝트를 특별한 저장소에 보관해두고 이후에 호출되는 메소드에서 저장된 connection을 가져다 쓰는 기법
 
 (그림 5-3 참고)
 
 트랜잭션 동기화 저장소는 작업 스레드마다 독립적으로 connection 오브젝트를 저장하고 관리하기 때문에 다중 사용자를 처리하는 서버의 멀티스레드 환경에서도 충돌이 날 염려가 없다.
 트랙잭션 동기화 기법을 사용하면 파라미터를 통해 일일이 Connection 오브젝트를 전달할 필요가 없게된다.
 
 
 트랜잭션 동기화 적용
 
스프링은 JdbcTemplate과 더불어 이런 트랜잭션 동기화 기능을 지원하는 유틸리티 메소드를 지원한다.

```java
public class UserService {
  private DataSource dataSource;
  ...
  public void setDataSource(DataSource dataSource) {
    this.dataSource = dataSource;
  }

  public void upgradeLevels() throws Exception {
    TransactionSynchronizationManager.initSynchronization(); // 트랜잭션 동기화 관리자를 이용해 동기화 작업을 초기화 한다.
    Connection c = DataSourceUtils.getConnection(dataSource); // DB 커넥션을 생성하고 트랜잭션을 시작한다.
    c.setAutoCommit(false);
    try{
      List<User> users = userDao.getAll();
      for(User user : users) {
        if (canUpgradeLevel(user)) {
          upgradeLevel(user);
        }
      }
      c.commit(); // 정상적으로 여기까지 수행되면 커밋
    } catch(Exception e) {
      c.rollback(); // 예외가 발생하면 롤백
      throw e;
    } finally {
      DataSourceUtils.releaseConnection((c, dataSource)); // 스프링 유틸리티 메소드를 이용해 DB 커넥션을 안전하게 닫는다.
      TransactionSynchronizationManager.unbindResource(this.dataSource);  // 동기화 작업 종료 및 정리
      TransactionSynchronizationManager.clearSynchronization();
    }
  }
  ...
}
``` 

스프링이 제공하는 트랜잭션 동기화 관리 클래스는 TransactionSynchronizationManager 이다. 이 클래스를 이용해 먼저 트랜잭션 동기화 작업을 초기화하도록 한다.
DataSource에서 직접 connection을 가져오지 않고 스프링이 제공하는 유틸리티 메소드인 DataSourceUtils.getConnection 을 사용하는 이유는 connection 오브젝트를 생성해줄 뿐만 아니라 트랜잭션 동기화에 사용하도록 저장소에 바인딩 해주기 때문이다.
작업중에는 connection을 공유하게되고 connection의 autoCommit = false 이므로 같은 트랜잭션을 공유한다.
작업이 끝나면 스프링 유틸리티 메소드의 도움을 받아 커넥션을 닫고 트랜잭션 동기화를 마치도록 요청한다.

=> JDBC의 트랜잭션 경계설정 메소드를 사용해 트랜잭션을 이용하는 전형적인 코드에 간단한 트랜잭션 동기화 작업만 붙여줌으로써, 지저분한 Connection 파라미터의 문제를 해결했다.


트랜잭션 테스트 보완

테스트용으로 확장해서 만든 TestUserSerivce는 UserService의 서브클래스이므로 UserService와 마찬가지로 DataSource를 DI 해줘야 한다.

```java
@Autowired
DataSource dataSource;

...

@Test
public void upgradeAllOrNothing() throws Exception {
  UserService testUserService = new TestUserService(users.get(3).getId());
  testUserService.setUserDao(this.userDao); 
  testUserService.setDataSource(this.dataSource);

  ...
```

JdbcTemplate과 트랜잭션 동기화

JdbcTemplate은 트랜잭션 동기화 저장소에 등록된 DB 커넥션이나 트랙잭션이 없는 경우에는 직접 DB 커넥션을 만들고 트랜잭션을 시작해서 JDBC 작업을 하지만 트랙잭션 동기화를 미리 시작해뒀다면 그때부터 실행되는 JdbcTemplate의
메소드에서는 직접 DB 커넥션을 만드는 대신 트랜잭션 동기화 저장소에 있는 DB 커넥션을 이용한다.
트랜잭션 적용 여부에 맞춰 동작하는 것이 JDBC 코드의 try/catch/finally 작업 흐름 지원, SQLException의 예외 변환과 함께 제공해주는 유용한 기능 중 하나이다.


트랜잭션 서비스 추상화

지금까지 만들어온 UserService, UserDao, UserDaoJdbc는 JDBC API를 잘 활용하고 트랜잭션을 적용했으면서도 책임과 성격에 따라 데이터 액세스 부분과 비지니스 로직을 잘 분리, 유지할 수 있게 만든 코드다.

기술과 환경에 종속되는 트랜잭션 경계설정 코드

하나의 트랜잭션 안에서 여러개의 DB에 데이터를 넣는 작업을 해야 할 필요가 발생헀다. 한 개 이상의 DB로의 작업을 하나의 트랜잭션으로 만드는 건 JDBC Connection을 이용한 트랜잭션 방식인 로컬 트랜잭션으로는 불가능하다.
로컬 트랜잭션은 하나의 DB Connection에 종속되기 때문이다. 따라서 각 DB가 독립적으로 만들어지는 Connection을 통해서가 아니라 별도의 트랜잭션 관리자를 통해 트랜잭션을 관리하는 글로벌 트랜잭션 방식을 사용해야 한다.
글로벌 트랜잭션을 적용해야 트랜잭션 매니저를 통해 여러 개의 DB가 참여하는 작업을 하나의 트랜잭션으로 만들 수 있다.

자바는 글로벌 트랜잭션을 지원하는 매니저를 지원하기 위한 API인 JTA를 제공하고 있다.
(그림 5-4)
어플리케이션에서는 기존의 방법대로 JDBC, JMS 등을 사용해서 필요한 작업을 수행한다. 단 트랜잭션은 JDBC나 JMS를 사용하여 제어하지 않고 JTA를 통해 트랜잭션 매니저가 관리하도록 위임한다.
JTA를 이용해 트랜잭션 매니저를 활용하면 여러 개의 DB나 메시징 서버에 대한 작업을 하나의 트랜잭션으로 통합하는 분산 트랜잭션 또는 글로벌 트랜잭션이 가능해진다.

JTA를 적용한 트랜잭션 처리 코드의 전형적인 구조는 아래와 같다.
```java
InitialContext ctx = new InitialContext();
UserTransaction tx = (UserTransaction)ctx.lookup(USER_TX_JNDI_NAME);

tx.begin();
Connection c = dataSource.getConnection();
try {
  tx.commit();
} catch (Exception e) {
  tx.rollback();
  throw e;
} finally {
  c.close();
}
```
전체적인 구조는 JDBC를 이용했을때와 유사하지만 문제는 JTA를 이용하는 글로벌 트랜잭션으로 바꾸려면 UserService의 코드를 수정해야 한다.
또 하이버네이트를 이용한 트랜잭션 관리 코드는 JDBC나 JTA의 코드와는 다르다 

문제점 : UserDao가 DAO 패턴을 사용해 구현 데이터 엑세스 기술을 유연하게 바꿔서 사용할 수 있게 했지만 UserService에서 트랜잭션 경계 설정을 해야할 필요가 생기면서 다시 특정 데이터 액세스 기술에 종속되는 구조가 되고말았다.


트랜잭션 API의 의존관계 문제와 해결책

원래 UserService는 UserDao 인터페이스에만 의존하는 구조였다. 그래서 DAO 클래스의 구현 기술이 JDBC에서 하이버네이트나 여타 기술로 바뀌더라도 UserService 코드는 영향을 받지 않았다. (OCP)
문제는 JDBC에 종속적인 Connection을 이용한 트랝개션 코드가 UserService에 들어오면서 UserDaoJdbc에 간접적으로 의존하는 코드가 되었다.
(그림5-5)

UserService의 메소드 안에서 트랜잭션 경계설정 코드를 제거할수는 없지만 특정 기술에 의존적인 Connection, UserTransaction, Session/Transaction API에 종속되지 않게 할 수 있는 방법은 있다.
트랜잭션 경계설정은 담당하는 코드는 일정한 패턴을 가지고 있으므로 추상화를 고려해볼 수 있다. JDBC, JTA, JPA, JDO 등등은 트랜잭ㄱ션 개념을 갖고 있으니 모드 트랜잭션 경계설정 방법에서 공통점이 있을 것이다.
이 공통적인 특징을 모아서 추상화된 트랜잭션 관리 계층을 만들어 UserService코드에서 추상화된 계층에만 의존하게 변경하자.


스프링의 트래잭션 서비스 추상화

스프링은 트랜잭션 기술의 공통점을 담은 트랜잭션 추상화 기술을 제공하고 있다. 이를 이용하면 어플리케이션에서 직접 각 기술의 트랜잭션 API를 이용하지 않고도 일관된 방식으로 트랜잭션을 제어하는 트랜잭션 경계설정 작업이 가능해진다.
(그림 5-6)

```java
public void upgradeLevels() throws Exception {
  PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource); // PlatformTransactionManager 인터페이스의 JDBC 트랜잭션 추상 오브젝트 생성
  TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
  try{
    List<User> users = userDao.getAll();
    for(User user : users) {
      if (canUpgradeLevel(user)) {
        upgradeLevel(user);
      }
    }
    transactionManager.commit(status);
  } catch(Exception e) {
    transactionManager.rollback(status);
    throw e;
  }
}
```

스프링이 제공하는 트랜잭션 경계썰정을 위한 추상 인터페이스는 PlatformTransactionManager 이다.
JDBC의 로컬 트랜잭션을 이용한다면 PlatformTransactionManager를 구현한 DataSourceTransactionManager를 사용하면 된다.


트랜잭션 기술 설정의 분리

트랜잭션 추상화 API를 적용한 UserService 코드를 JTA를 이용하는 글로벌 트랜잭션으로 변경하려면 => PlatformTransactionManager 구현 클래스를 DataSourceTransactionManager 에서 JTATransactionManager로 변경하기만 하면된다.
UserDao를 하이버네이트로 구현했다면 HibernateTransactionManager를, JPA를 적용했다면 JPATransactionManager를 사용하면 된다. 모두 PlatformTransactionManager 인터페이스를 구현한 것이니 트랜잭션 경계설정을 위한 getTransaction, commit, rollback 메소드를 수정할 필요가 없다.

문제점 : 어떤 트랜잭션 매니저 구현 클래스를 사용할지 UserService 코드가 알고 있는 것은 DI 원칙에 위배된다. 자신이 사용할 구체적인 클래스를 스스로 결정하고 생성하지 말고 컨테이너를 통해 외부에서 제공받게 하는 DI 방식으로 바꾸자.

```java
public class UserService {
  private UserDao userDao;
  
  private PlatformTransactionManager transactionManager;
  
  public void setTransactionManager(PlatformTransactionManager transactionManager){ // DI를 위한 수정자
    this.transactionManager = transactionManager;
  }

  public void setUserDao(UserDao userDao) {
    this.userDao = userDao;
  }

  public void upgradeLevels() throws Exception {
  TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try{
      List<User> users = userDao.getAll();
      for(User user : users) {
        if (canUpgradeLevel(user)) {
          upgradeLevel(user);
        }
      }
      this.transactionManager.commit(status);
    } catch(Exception e) {
      this.transactionManager.rollback(status);
      throw e;
    }
  }
  ...
```
```xml
<bean id="userService" class="com.example.demo.user.service.UserService">
    <property name="userDao" ref="userDao"/>
    <property name="transactionManager" ref="transactionManager"/>
</bean>

<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```
DataSourceTransactionManager 는 dataSource 빈으로부터 Connection을 가져와 트랜잭션 처리를 해야하기 때문에 dataSoruce 프로퍼티를 갖는다.
userService 빈도 기존의 dataSource 프로퍼티를 없애고 새롭게 추가한 transactionManager 빈을 DI 받도록 설정했다.

테스트도 upgradeAllOrNoting 메소드는 한 트랜잭션으로 수행되어야 테스트가 성공하기 때문에 testService.setTrnasactionManager로 DI 해줘야 한다.

만약 UserService의 트랜잭션을 JDBC에서 JTA로 고치고 싶다면 tansactionManager 빈 설정만 아래와 같이 고치면 된다.

```xml
<bean id="userService" class="com.example.demo.user.service.UserService">
    <property name="userDao" ref="userDao"/>
    <property name="transactionManager" ref="transactionManager"/>
</bean>

<bean id="transactionManager" class="org.springframework.transaction.jta.JtaTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

마찬가지로 DAO를 하이버네이트, JPA, JDO 등등 다른것을 사용하도록 수정했다면 그에 맞게 transactionManger의 구현클래스만 변경해주면 된다. UserService의 코드는 조금도 수정할 필요가 없다.

5-3 서비스 추상화와 단일 책임 원칙
----------------------------------------------------------------------------

수직, 수평 계층구조와 의존관계

기술과 서비스에 대한 추상화기법을 이용하면 특정 기술환경에 종속되지 않는 코드를 만들 수 있다. 
UserDao와 UserService는 각각 담당하는 코드의 기능적인 관심에 따라 분리되고, 서로 불필요한 영향을 주지 않으면서 독자적으로 확장이 가능하도록 만든 것이다. 같은 어플리케이션 로직을 담고 있는 코드지만 내용에 따라 분리했기 때문에 수평적인 분리라고 볼 수 있다.
트랜잭션의 추상화는 이와 좀 다르다. 어플리케이션 비지니스 로직과 하위에서 동작하는 로우레벨 트랜잭션 기술이라는 아예 다른 계층의 특성을 갖는 코드를 분리한 것이다.

(그림 5-7)

어플리케이션 로직의 종류에 따른 수평적인 구분이든, 로직과 기술이라는 수직적인 구분이든 모두 결합도가 낮으며, 서로 영향을 주지 않고 자유롭게 확장될 수 있는 구조를 만들 수 있는 데는 스프링의 DI가 중요한 역할을 하고 있다.
DI의 가치는 이렇게 관심, 책임, 성격이 다른 코드를 깔끔하게 분리하는 데 있다.

단일 책임 원칙(SRP)

단일 책임 원칙은 하나의 모듈은 한가지 책임을 가져야 한다는 의미 (= 하나의 모듈이 바뀌는 이유는 한 가지여야 한다)

단일 책임 원칙의 장점

단일 책임 원칙을 잘 지키고 있다면 어떤 변경이 필요할 때 수정 대상이 명확해진다. 
ex) 기술이 바뀌면 기술 계층과의 연동을 담당하는 기술 추상화 계층의 설정만 바꿔주면 된다.  데이터를 가져오는 테이블 이름이 바뀌었다면 데이터 엑세스 로직을 담고 있는 UserDao를 변경하면 된다.

비지니스로직의 변경과 기술적인 구현이 한  계층에 있으면 둘중하나의 변경이 있을때마다 코드를 수정해줘야하고 Dao나 Service가 N개가 있는 케이스에서는 수정해줘야할 클래스의 수가 N배 만큼 많아진다.
=> 절절하게 책임과 관심이 다른 코드를 분리하고, 서로 영향 주지 않도록 추상화 기법을 도입하고, 어플리케이션 로직과 기술/환경을 분리하는 등의 작업은 갈수록 복잡해지는 엔터프라이즈 어플리케이션에는 반드시 필요하다. 이를 위한 핵심적인 도구가 바로 스프링이 제공하는 DI 이다.

객체지향 설계와 프로그래밍 원칙은 서로 긴밀하게 관련이 있다. 
=>SRP를 잘 지키는 코드를 만드려면 인터페이스를 도입하고  이 사이를 DI로 연결해야하며 그 결과로 SRP뿐만 아니라 OCP도 잘 지키게 되고, 모듈 사이 결합도가 낮아 서로의 변경이 영향을 주지 않고 같은 이유로 응집도도 높아진다. 그러면서 자연스럽게 다양한 디자인 패턴들을 적용하게 된다
=> 스프링의 의존관게 주입 기술인 DI는 모든 스프링 기술의 기반이 되는 핵심 엔진이자 원리이며, 스프링이 지지하고 지원하는 좋은 설계과 코드를 만드는 코든 과정에서 사용되는 중요한 도구다.

5-4 메일 서비스 추상화
----------------------------------------------------------------------------

사용자 레벨 업그레이드시 대상 사용자에게 안내 메일을 발송하는 기능을 추가하자 => User에 email필드를 추가하고 UserService에 upgradeLevel에 메일발송 기능을 담당하는 메소드를 호출하게 한다.

JavaMail을 이용한 메일 발송 기능

- DB - User 테이블 - email 컬럼 추가
- User VO에 email 필드 추가, 생성자 수정,
- UserDa의 userMapper, insert, update에 email 필드 반영 
- UserDaoTest 코드 수정

JavaMail 발송

```java
protected void upgradeLevel(User user) {
  user.upgradeLevel();
  userDao.update(user);
  sendUpgradeEMail(user);
}
```
```java
private void sendUpgradeEMail(User user) {
	Properties props = new Properties();
	props.put("mail.smtp.host", "mail.ksug.org");
	Session s = Session.getInstance(props, null);

	MimeMessage message = new MimeMessage(s);
	try {
		message.setFrom(new InternetAddress("useradmin@ksug.org"));
		message.setRecipient(Message.RecipientType.TO, new InternetAddress(user.getEmail()));
		message.setSubject("Upgrade 안내");
		message.setText("사용자님의 등급이 " + user.getLevel().name() + "로 업그레이드되었습니다.");

		Transport.send(message);
	} catch (AddressException e) {
		throw new RuntimeException(e);
	} catch (MessagingException e) {
		throw new RuntimeException(e);
	}
}
```


JavaMail이 포함된 코드의 테스트

메일 서버가 준비되어 있지 않으면?=> 테스트 실패
테스트하면서 매번 메일이 발송되는것이 바람직한가? => 운영 메일서버에 부하를 줄 수 있으므로 바람직하지 않다.

메일 발송 기능은 보조적인 기능에 불과하고 엄밀히 말하면 테스트 불가능하다. 
=> 하지만 메일 서버는 충분히 테스트된 시스템이기 때문에 SMTP로 메일 전송 요청을 받으면 문제없이 메일을 전송할 것이다. (단지 JavaMail과 연동해서 메일 전송 요청을 받는 것까지만 테스트) 
=> 마찬가지로 JavaMail을 자바 표준기술이고 수많은 시스템에서 사용되어 검증되었기 때문에 JavaMail로 메일전송 요청을 하기만 하면 메일을 전송할 것이다. (JavaMail로 메일 전송 요청을 받는것 까지만 테스트)
=> 개발 중이거나 테스트 수행할 때는 JavaMail을 대신할 수 있지만 JavaMail과 인터페이스가 같은 코드가 동작되도록 만들자


테스트를 위한 서비스 추상화

하지만 JavaMail은 Session 오브젝트를 만들어야만 메일메세지를 생성할수 있고 Session은 인터페이스가 아닌 클래스이며 생성자가 private으로 되어있어 상속도 불가능하다 
=> JavaMail의 구현을 테스트용으로 바꿔치기하는것은 불가능 
=> JavaMail처럼 테스트하기 힘든 구조인 API를 테스트하기 좋게 만들기 위해 서비스 추상화를 적용하면 된다.

```java
public interface MailSender {
  void send(SimpleMailMessage simpleMessage) throws MailException;
  void send(SimpleMailMessage[] simpleMessages) throws MailException;
}
```

```java
private void sendUpgradeEMail(User User){
  JavaMailSenderImpl mailSender = new JavaMailSenderImpl(); // MailSender 구현 클래스(JavaMailSenderImpl)의 오브젝트를 생성한다.
  mailSender.setHost("mail.server.com");

  SimpleMailMessage maiMessage = new SimpleMailMessage();
  mailMessage.setTo(user.getEmail());
  mailMessage.setFrom("useradmin@ksug.org");
  mailMessage.setSubject("Upgrade 안내");
  mailMessage.setText("사용자님의 등급이 " + user.getLvl().name() + "로 업그레이드되었습니다.");

  this.mailSender.send(mailMessage);
}
```

JavaMail API를 사용하는 JavaMailSenderImpl 클래스의 오브젝트를 코드에서 직접 사용하기 때문에 아직 테스틑용 오브젝트로 대체할 수 없다 => DI를 적용하여 JavaMailSenderImpl 클래스가 구현한 MailSender 인터페이스만 남기고 구체적인 메일 전송 구현을 담은 클래스의 정보는 코드에서 제거한다.

```java
private MailSender mailSender;

public void setMailSender(MailSender mailSender){
  this.mailSender = mailSender;
}

public void sendUpgradeEmail(User user) {
  SimpleMailMessage mailMessage = new SimpleMailMessage();
  mailMessage.setTo(user.getEmail());
  mailMessage.setFrom("sodongyocs@gmail.com");
  mailMessage.setSubject("NOTIFY LEVEL-UP");
  mailMessage.setText("Your level became " + user.getLevel().name());

  this.mailSender.send(mailMessage);
}
```


테스트용 메일 발송 오브젝트

이제 테스트를 실행하면 JavaMail API를 직접 사용했을때와 동일하게 지정된 메일 서버로 메일이 발송된다.
=> 우리가 정말 원하는 건 JavaMail을 사용하지 않고 메일 발송기능이 포함된 코드를 테스트 하는 것이기 때문에 메일 전송기능을 추상화해서 인터페이스를 적용하고 DI를 통해 빈으로 분리해두었다
=> 스프링이 제공한 메일 전송기능에 대한 인터페이스가 있으니 이를 구현해서 테스트용 메일 전송 클래스를 만들어보자. (실제로는 아무것도 하지 않는 MailSender 구현 클래스를 만들자)

```java
public class DummyMailSender implements MailSender {
  public void send(SimpleMailMessage simpleMessage) throws MailException {
  // do nothing
  }
  
  public void send(SimpleMailMessage[] simpleMessages) throws MailException {
  // do nothing
  }
}
```

DummyMailSender는 MailSender 인터페이스를 구현했을 뿐 하는일이 없다. 테스트 설정파일의 mailSender 빈 클래스를 다음과 같이 JavaMail을 사용하는 JavaMailSender 대신 DummyMailSender로 변경한다.

```xml
<bean id="mailSender" class="springbook.user.service.DummyMailSender"  />
```


 테스트와 서비스 추상화
 
 일반적으로 서비스 추상화라고 하면 트랜잭션과 같이 기능은 유사하나 사용 방법이 다른 로우레벨의 다양한 기술에 대해 추상 인터페이스와 일관성 있느 접근 방법을 제공해주는 것을 말한다. 반면에 JavaMail의 경우처럼 테스트를 어렵게 만드는 건전하지 않은 방법으로 설계된 API를 사용할 때도 유용하게 쓰인다.
 (그림 5-10)
 
 JavaMail이 아닌 다른 메세징 서버의 API를 이용해 메일을 전송해야 하는 경우가 생겨도 해당 기술의 API를 이용하는 MailSender 구현 클래스를 만들어서 DI 해주면 된다.
 메일을 바로 전송하지 않고 큐에 담아뒀다가 일괄적으로 발송하는 기능을 만들때도 MailSender로 추상화된 메일 전송 추상화 계층이 도움된다. 
 
 서비스 추상화란 원할한 테스트만을 위해서도 충분한 가치가 있다. 기술이나 환경이 바뀔 가능성이 있음에도 JavaMail 처럼 확장이 불가능하게 설계해놓은 API를 사용해야하는 경우라면 추상화 계층의 도입을 고려해야한다, 
 특별히 외부 리소스와 연동되는 대부분 작업은 추상화 할 수 있다.
 
 
 테스트 대역
 
 테스트할 대상이 의존하고 있는 오브젝트를 DI를 통해 바꿔치기하기 한다.
 
 
 의존 오브젝트 변경을 통한 테스트 방법
 
(그림 5-11)
테스트에서는 운영 DB의 연결도 WAS의 DB 폴링 서비스의 사용도 번거로울 뿐이다. UserDaoTest의 관심은 UserDao가 어떻게 동작하느냐 이지 그 뒤에 존재하는 DB 커넥션 풀이나 DB 자체에 있지 않다.
그래서 이를 대신할 수 있도록 테스트환경에서도 잘 동작하고 준비 과정도 간단한 DataSource를 사용하고 DB도 개발자 PC에 설치해서 사용해도 무방한 버전을 이용하게 한 것이다.

(그림 5-12)
테스트 때는 JavaMailSenderImpl와 JavaMail을 통한 메일 서버로 이어지는 구성이 오히려 손해다. UserDaoTest의 관심사는 UserService에서 구현해놓은 사용자 정보를 가공하는 비지니스 로직이지 메일이 어떻게 전송되느냐는 관심없다.
테스트 대상이 되는 코드를 수정하지 않고, 메일 발송 작업 때문에 UserService자체에 대한 테스트에 지장을 주지 않기 위해 도입한 것이 DummyMailSender 이다.

테스트 대상이 되는 오브젝트가 다른 오브젝트에 의존하는 일은 매우 흔하다. 테스트 대상인 오브젝트가 의존 오브젝트를 갖고 있기 때문에 발생하는 테스트상의 문제점들이 있다
간단한 오브젝트의 코드를 테스트하는데 너무 거창한 작업이 뒤따르는 것이다. 이럴때 아예 아무일도 하지 않는 빈 오브젝트로 대체해주는 것이 해결책이고 DI가 유용하다.


테스트 대역의 종류와 특징

- test double(테스트 대역) : 테스트 환경을 만들어주기 위해 테스트 대상이 되는 오브젝트의 기능에만 충실하게 수행되면서 자주 테스트를 실행할 수 있도록 사용되는 오브젝트
- test stub(테스트 스텁) : 테스트 대역의 일종으로, 테스트 대상 오브젝트의 의존객체로서 존재하면서 테스트 동안에 코드가 정상적으로 수행할수 있도록 돕는 것

많은 경우 테스트 스텁이 결과를 돌려줘야 할 때도 있다. 이럴 땐 스텁에 미리 테스트중에 필요한 정보를 리턴해주도록 만들 수 있다. 또는 어떤 스텁은 메소드를 호출하면 강제로 예외를 발생시키게 해서 테스트 대상 오브젝트가 예외상황에서 어떻게 반응하지는지 테스트 할 때 적용할 수도 있다.

테스트 대상 오브젝트의 메소드가 돌려주는 결과 뿐 아니라 테스트 오브젝트가 간접적으로 의존 오브젝트에 넘기는 값과 그 행위 자체에 대해서도 검증하고싶다면 이를 위해 특별히 설계된 mock object(목 오브젝트)를 사용해야 한다.
목 오브젝트는 스텁처럼 테스트 오브젝트가 정상적으로 실행되도록 도와주면서 테스트 오브젝트와 자신의 사이에서 일어나는 커뮤니케이션 내용을 저장해뒀다가 테스트 결과를 검증하는 데 활용할 수 있게 해준다.

테스트 대상이 넘겨주는 출력 값을 보관해두는 기능을 추가한 MockMailSender를 만들어보자
```java
public class MockMailSender implements MailSender {
  private List<String> requests = new ArrayList<>();

  public List<String> getRequests() {
    return requests; 
  }

  public void send(SimpleMailMessage mailMessage) throws MailException {
      requests.add(mailMessage.getTo()[0]);
  }

  public void send(SimpleMailMessage[] mailMessages) throws MailException {
  }
}
```

```java
public class UserServiceTest {
  @Test
  @DirtiesContext   // 컨텍스트의 DI 설정을 변경하는 테스트라는 것을 알려준다.
  public void upgradeLvls() throws Exception {
    userDao.deleteAll();
    for (User user : users)  userDao.add(user);
    
    // 메일 발송 결과를 테스트할 수 있도록 목 오브젝트를 만들어 userService 의존 오브젝트로 DI
    MockMailSender mockMailSender = new MockMailSender();
    userService.setMailSender(mockMailSender);

    // 업그레이드 테스트 메일 발송이 일어나면 MockMailSender 오브젝트의 리스트에 그 결과가 저장됨
    userService.upgradeLevels();

    checkLevelUpgraded(users.get(0), false);
    checkLevelUpgraded(users.get(1), true);
    checkLevelUpgraded(users.get(2), false);
    checkLevelUpgraded(users.get(3), true);
    checkLevelUpgraded(users.get(4), false);

    // 목 오브젝트에서 저장한 메일 수신자 목록을 가져와 업그레이드 대상과 일치하는 지 확인
    List<String> request = mockMailSender.getRequests();
    assertThat(request.size(), is(2));
    assertThat(request.get(0), is(users.get(1).getEmail()));
    assertThat(request.get(1), is(users.get(3).getEmail()));
  }
}
```

목 오브젝트를 이용한 테스트라는게 작성하기에는 간단하지만 기능은 상당히 강하다는 것을 알 수 있다. 
보통의 테스트 방법으로는 검증하기 까다로운 테스트 대상 오브젝트 내부에서 일어나는 일이나 다른 오브젝트 사이에서 주고받는 정보까지 쉽게 검증할 수 있기 때문이다.

5-5 정리
----------------------------------------------------------------------------
   * 비지니스 로직을 담은 코드는 데이터 엑세스 로직을 담은 코드와 분리되는 것이 바람직하다. 비지니스 로직 코드 또한 내부적으로 책임과 역할에 따라 메소드로 정리돼야 한다.
   * 이를 위해서 DAO의 기술 변화에 서비스 계층의 코드가 영향을 받지 않도록 인터페이스와 DI를 잘 활용해서 결합도를 낮춰야 한다.
   * DAO를 사용하는 비지니스 로직에는 단위 작업을 보장해주는 트랜잭션 적용이 필요하다
   * 트랜잭션의 시작과 종료를 지정하는 일을 트랜잭션 경계설정이라고 하고 이는 주로 비지니스 로직 안에서 일어나는 경우가 많다.
   * 시작된 트랜잭션 정보를 담은 오브젝트를 파라미터로 DAO에 전달하는 방법은 매우 비효율적이기 때문에 스프링이 제공하는 트랜잭션 동기화 기법을 활용하는 것이 좋다.
   * 자바에서 사용되는 트랜잭션 API의 종류와 방법은 다양하다. 환경과 서버에 따라 트랜잭션 방법이 변경되면 경계썰정 코드도 함께 변경돼야 한다.
   * 트랜잭션 방법에 따라 비지니스 로직을 담은 코드가 함께 변경되면 SRP에 위배되며 DAO가 사용하는 특정 기술에 대해 강한 결합을 만들어 낸다
   * 트랜잭션 경게설정 코드가 비지니스로직 코드에 영향을 주지 않게 하려면 스프링이 제공하는 트랜잭션 서비스 추상화를 이용하면 된다.
   * 서비스 추상화는 로우레벨의 트랜잭션 기술과 API의 기술변화와 상관없이 일관된 API를 가진 추상화 계층을 도입한다.
   * 서비스 추상화는 테스트하기 어려운 JavaMail과 같은 기술에도 적용할 수 있다. 테스트를 편리하게 작성하도록 도와주는 것만으로도 서비스 추상화는 가치가 있다.
   * 테스트 대상이 사용하는 의존 오브젝트를 대체할 수 있도록 만든 오브젝트를 테스트 대역이라고 한다.
   * 테스트 대역은 테스트 대상 오브젝트가 원할하게 동작할 수 있도록 도우면서 테스트를 위해 간접적인 정보를 제공해주기도 한다.
   * 테스트 대역 중에서 태스트 대상으로부터 전달받은 정보를 검증할 수 있도록 설계된 것을 목 오브젝트라고 한다.
   