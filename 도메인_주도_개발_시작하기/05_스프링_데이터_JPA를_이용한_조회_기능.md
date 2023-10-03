# Chapter 5 : 스프링 데이터 JPA를 이용한 조회 기능

- [검색을 위한 스펙](#검색을-위한-스펙)
  - [스펙 조합](#스펙-조합)
- [JPA를 위한 스펙 구현](#JPA를-위한-스펙-구현)
- [조회 전용 기능 구현](#조회-전용-기능-구현)
- [💡 생각해볼 점](#-생각해볼-점) 

<br/>

## 검색을 위한 스펙     
- repository에서 aggregate를 찾을 때, `식별자를 이용`하는 것이 기본이지만 `식별자 외에 다양한 조건으로` aggregate를 찾아야 할 때가 있음
  
  - ex) `검색 조건이 다양`할 경우
    
    => `스펙(Specification)` 이용하기

- `Specification` : 검색 조건을 다양하게 조합해야 할 때 사용
  
  - aggreagate가 특정 조건을 충족하는지 검사할 때 사용하는 interface   

<br/>

- ex)

```java
public interface Specification<T> {
	public boolean isSatisfiedby(T agg);
}

public class OrdererSpec implements Specification<Order> {
	private String ordererId;

	public boolean isSatisfiedBy(Order agg) {
		return agg.getOrdererId().getMemberId().getId().equals(ordererId);
	}
}
```

==> client는 repository에 전달해주기만 하면 됨   

```java
Specification<Order> ordererSpec = new OrdererSpec("madvirus");
List<Order> orders = orderRepository.findAll(ordererSpec);
```

<br/>

### 스펙 조합
- 두 스펙을 `AND 연산자나 OR 연산자로 조합`해서 새로운 스펙을 만들 수 있고, 조합한 스펙을 다시 조합해서 더 복잡한 스펙을 만들 수 있음

```java
Specification<Order> ordererSpec = new OrdererSpec("madvirus");
Specification<Order> ordererDateSpec = new OrdererDateSpec(fromDate, toDate);
AndSpec<T> spec = new AndSpec(ordererSpec, orderDateSpec);

List<Order> orders = orderRepository.findAll(spec);
```

<br/>

- 위의 repository code는 `모든 aggregate를 조회한 다음`에 `스펙을 이용해서 걸러내는` 방식
     
  => `성능상 문제`가 있음
  
  => 실제 구현에서는 `쿼리의 where 절에 조건 붙여서` 필요한 데이터를 걸러야 함

<br/>

## JPA를 위한 스펙 구현
- JPA는 `다양한 검색 조건을 조합`하기 위해 `CreteriaBuilder`와 `Predicate` 사용   
  
  => JPA를 위한 스펙은 CriteriaBuilder와 Predicate를 이용해서 검색 조건 구현해야 함   

```java
public interface Specification<T> {
	Predicate toPredicate(Root<T> root, CriteriaBuilder db);
}

public class OrdererSpec implements Specification<Order> {
	private String ordererId;

	public Predicate toPredicate(Root<Order> root, CriteriaBuilder cb) {
		return cb.equal(root.get(Order_.orderer)
				.get(Orderer_.memberId).get(MemberId_.id),ordererId);
	}
}
```

<br/>

## 조회 전용 기능 구현
- repository는 aggregate의 저장소를 표현하는 것

  => 다음 용도로 repository를 사용하는 것은 적합하지 않음   

  - `여러 aggregate를 조합`해서 한 화면에 보여주는 데이터 제공

    - JPA의 지연 로딩과 즉시 로딩 설정, 연관 매핑 설정 고려 필요   

    - aggregate 간 직접 연관을 맺는 경우, id 참조할 때의 장점을 활용할 수 없음   

  - 각종 `통계 데이터` 제공

    - 다양한 테이블을 조인하거나 DBMS 전용 기능 사용을 JPQL이나 Criteria 로 처리하기 힘듦
        
  ==> 애초에 이런 기능은 `조회 전용 쿼리로 처리해야` 하는 것들!  

<br>

- JPA, Hibernate를 사용하면 다음의 방법들로 조회 전용 기능을 구현 가능
  
  - `동적 인스턴스` 생성
 
    - JPA는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공함    
      -> JPQL을 그대로 사용하기에 지연/즉시 로딩과 같은 고민이 필요없이 데이터를 조회할 수 있음     
    
  - `hibernate의 @Subselect 확장` 기능
    
    - @Subselect는 쿼리 결과를 @Entity로 매핑할 수 있는 기능 
    
  - `native query` 이용

<br/>

## 💡 생각해볼 점

- 너무 복잡해 보이는데, 굳이 JPA Criteria를 사용해야할까???
  
  - `JPQL` : 쿼리 실행 전까지는 문법오류 판단 못함
    
  - `Criteria` : 컴파일 단계에서 문법오류 판단 가능 but 가독성이랑 코드 복잡성 무슨일,,
    
  - `QueryDSL` : 컴파일 단계에서 문법오류 판단 가능 + 쿼리처럼 사용 가능
 
    => 나는 QueryDSL 사용해야지 ㅎㅎ!   

<br/>

- 몇주전에 회사에서 통계 데이터를 써야하는 기능을 개발해야해서 난감한 적이 있었다(내가 개발하는 내용은 아니었지만)      
  여러 테이블의 조인이 필요한 상황인데 인덱스를 걸고 이것저것 다 해봐도 성능이 나오지 않았다    
  다행히 통계쪽에서 사용하고 있는 데이터가 있어서 그쪽에서 데이터를 끌고오는 크론작업을 통해 마무리 지었는데   
  책을 읽다보니 더 좋게 해결할 수 있는 방법이 있을까? 하면서 갑자기 생각났다...뜬금.. 

<br/>

- 내가 이해한 5장
  - 조회화면을 위해 별도로 조회 전용 모델과 DAO를 만드는 내용

<br/>
