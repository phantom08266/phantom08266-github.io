---
title: Mybatis MapperId
date: 2023-10-23
categories: [ Spring, mybatis ]
tags: [ mybatis, mapperId ]
math: true
mermaid: true
---


사내에서 운영환경을 배포한 뒤 로그를 모니터링 하던 도중 갑자기 IllegalArgumentException이 발생하였다. <br>
원인을 파악해보니 MapperId가 Ambiguous하다는 것이었다. 이번 기회에 Mybatis가 어떻게 MapperId를 관리하는지, 
무엇이 문제였는지 알아보고 정리하고자 한다.

![image1](https://github.com/phantom08266/phantom08266.github.io/assets/39672033/482a3fc2-7fdc-4ba5-b746-b4f6e0bbee22)

## Mybatis는 어떻게 MapperId를 관리하고 있을까?

우선 디버깅을 해보니 최종적으로 org.apache.ibatis.session.Configuration 클래스에서 Map 형식으로 MapperId를 관리하는 것으로 파악했다. <br>
그리고 MapperId를 관리하는 Map은 아래와 같이 구성되어 있다.

```java
Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")
      .conflictMessageProducer((savedValue, targetValue) ->
          ". please check " + savedValue.getResource() + " and " + targetValue.getResource());
```
코드를 보면 StrictMap 클래스로 초기화를 하고 있는데, 이 클래스는 HashMap을 상속받아 구현한 커스텀 클래스이다. <br>
StrictMap 클래스는 put 메서드를 Override하고 있는데 해당 메서드의 코드는 아래와 같다.

```java
@Override
@SuppressWarnings("unchecked")
public V put(String key, V value) {
  if (containsKey(key)) {
    throw new IllegalArgumentException(name + " already contains value for " + key
        + (conflictMessageProducer == null ? "" : conflictMessageProducer.apply(super.get(key), value)));
  }
  if (key.contains(".")) {
    final String shortKey = getShortName(key);
    if (super.get(shortKey) == null) {
      super.put(shortKey, value);
    } else {
      super.put(shortKey, (V) new Ambiguity(shortKey));
    }
  }
  return super.put(key, value);
}
```
코드를 보니 에러로그에서 확인한 Ambiguity 객체를 put하는 것을 확인할 수 있다. <br>
또한 메서드가 종료하기전 key값 그대로 Map의 Key값으로 사용하는 것을 확인할 수 있다. <br>
즉 최종적으로 해당 Map에는 하나의 MapperId를 저장하는 것이 아니라 2개씩 저장하게 된다. 첫번째는 FullPackageName+MapeprId이 Key인 값,
두번째는 MapperId만 Key인 값이다. <br>
그렇다면 프로젝트에서 어떻게 사용했길래 Ambiguity를 Value로 가지고 있을까?🤔


### Ambiguity를 Value로 가지는 경우

원인은 개발한 프로젝트에서는 레거시 코드가 있다보니 @Mapper를 사용하는 부분만 있는것이 아니라 SqlSessionTemplate을 직접 주입받아서 사용하는 부분도 있었다. <br>
문제는 SqlSessionTemplate을 직접 주입받아 사용하는 부분에서 MapperId를 입력 시 FullPackageName+MapperId을 사용하지 않아서였다.
[공식문서](https://mybatis.org/spring/ko/sqlsession.html)에서도 아래와 같이 MapperId를 입력할 때 FullPackageName+MapperId를 사용하라고 나와있다.

```java
public class UserDaoImpl implements UserDao {

  private SqlSession sqlSession;

  public void setSqlSession(SqlSession sqlSession) {
    this.sqlSession = sqlSession;
  }

  public User getUser(String userId) {
    return sqlSession.selectOne("org.mybatis.spring.sample.mapper.UserMapper.getUser", userId);
  }
}
```

즉 공식문서 예시처럼 사용하지 않고 getUser로 파라미터를 넘기는 것이 문제였다. <br>
그런데 왜 Ambiguity 객체를 반환하는 것일까? <br>
아까 put할때 보았듯이 동일한 getUser라는 MapperId를 가진 2개의 객체가 Map에 저장되어 있기 때문이다. <br>
예를들어 test.getUser라는 MapperId를 가진 객체가 2개가 있다고 가정하면 2개의 데이터가 Map에 저장된다. 
- test.getUser
- getUser

이때 MapperScan이 다른 패키지에 있는 동일한 MapperId를 읽어와 put할 경우 문제가 발생한다. <br>
즉 example.getUser라른 값을 put하게되면 최종적으로 아래와 같이 Map에 적재될 것이다.
- test.getUser
- getUser
- example.getUser

다만 이때 getUser는 HashMap의 특성상 덮어쓰기가 되어 Ambiguity 객체가 저장되는 것이다. <br>


### 결론
@Mapper를 사용하면 조회 시 내부적으로 FullPackageName+MapperId를 사용해서 조회하기 때문에 문제가 없다. <br>
따라서 SqlSessionTemplate을 직접 주입받아 사용하기 보다는 @Mapper 나 @Repository를 사용하는 것이 좋다. <br>
만약 SqlSessionTemplate을 직접 주입받아 사용해야 한다면 MapperId를 입력할 때 FullPackageName+MapperId를 사용하도록 하자. <br>
