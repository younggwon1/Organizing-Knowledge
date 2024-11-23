# PathParam vs QueryParam

> 공부 이유로는 Path Parameter과 Query Parameter 의 차이점이 무엇인지? 그리고 언제 각각을 사용하는지를 알고 싶어 공부를 진행.



## Path Parameter

~~~Url
/users/123  # 아이디가 123인 사용자 찾기
~~~

- Spring RESTFul : @PathVariable

호출된 URL의 경로 부분에서 값을 읽는다.

Path Parameter 는 모든 Method 에서 사용할 수 있다.

- GET
- POST
- PUT
- DELETE



언제 사용?

어떤 **resource를 식별**하고 싶을 경우 **Path Parameter** 를 사용한다.



## Query Parameter

```Url
/users?title=developer&status=active  # 개발자 직함을 가진 현직 직원 찾기
```

- Spring RESTFul : @RequestParam

URI 호출의 QueryParameters 에서 값을 읽는다.

URL 에서 특정한 조건을 주고싶을 때 사용한다.

- 같은 API 이지만, 서로 다른 조건으로 나열하는 것이 필요한 상황에서 사용한다.
  - EX) 전체 POD 리스트에서 특정 NameSpace 만 따로 선택하여 처리할 때..

QueryParameters는 다음의 Method 에서 사용할 수 있다.

- GET
- DELETE



언제 사용?

**항목을 정렬하거나 필터링, 데이터 수 조절, 검색 등**을 하고 싶을 경우 **Query Parameter**를 사용한다.





참고 문서

1. [[API] Path parameter VS Query Parameter (기록용)](https://velog.io/@juno97/API-Path-parameter-VS-Query-Parameter-%EA%B8%B0%EB%A1%9D%EC%9A%A9)
2. [[Java/SpringBoot] @RequestParam vs @PathVariable 쓰임새, 사용법, 차이점](https://velog.io/@dongscholes/JavaSpringBoot-RequestParam-vs-PathVariable-%EC%93%B0%EC%9E%84%EC%83%88-%EC%82%AC%EC%9A%A9%EB%B2%95-%EC%B0%A8%EC%9D%B4%EC%A0%90)