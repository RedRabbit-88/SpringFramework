# 스프링 프레임워크 학습 (토비의 스프링 3.1)


## 1장 오브젝트와 의존관계
https://github.com/RedRabbit-88/SpringFramework/wiki/1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84#1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84


## 2장 테스트
https://github.com/RedRabbit-88/SpringFramework/wiki/2%EC%9E%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8#2%EC%9E%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8


## 3장 템플릿
https://github.com/RedRabbit-88/SpringFramework/wiki/3%EC%9E%A5-%ED%85%9C%ED%94%8C%EB%A6%BF#3%EC%9E%A5-%ED%85%9C%ED%94%8C%EB%A6%BF


## 4장 예외
https://github.com/RedRabbit-88/SpringFramework/wiki/4%EC%9E%A5-%EC%98%88%EC%99%B8#4%EC%9E%A5-%EC%98%88%EC%99%B8


## 5장 서비스 추상화

https://github.com/RedRabbit-88/SpringFramework/wiki/5%EC%9E%A5-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%83%81%ED%99%94#5%EC%9E%A5-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%83%81%ED%99%94


## 6장 AOP
https://github.com/RedRabbit-88/SpringFramework/wiki/6%EC%9E%A5-AOP#6%EC%9E%A5-aop


## 7장 스프링 핵심 기술의 응용

* 스프링의 3대 핵심 기술: **Ioc/DI, 서비스 추상화, AOP**


### 7.1 SQL과 DAO의 분리

* SQL 변경이 필요한 상황이 발생하면 SQL을 담고 있는 DAO 코드가 수정될 수 밖에 없음
<br>-> SQL을 적절히 분리해 DAO 코드와 다른 파일이나 위치에 두고 관리가 필요


### 7.1.1 XML 설정을 이용한 분리

* 가장 쉬운 방법은 SQL을 스프링 XML 설정파일로 분리하는 것
<br>-> 매번 새로운 SQL이 추가될 때마다 프로퍼티를 추가하고 DI를 위한 변수와 수정자 메서드도 만들어야 함.
```java
public class UserDaoJdbc implements UserDao {
  private String sqlAdd;
  
  public void setSqlAdd(String sqlAdd) {
    this.sqlAdd = sqlAdd;
	}
}
```
