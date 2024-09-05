# spring-landlog

## 자바 버전 : 17

## DDD 설계 구조, Session Login 이용

DDD, 네개의 Layer로 구성되어있다.
- presentation
- application
- domain
- infrastructure

presentation : controller, dto, router 등 Client와 직접적인 상호작용이 일어나는 Layer  
application : service 와 같은 applicaiton logic이 구현되는 Layer  
domain : 문제의 영역이 정의되는 Layer, application logic에서 사용되는 비즈니스 로직을 구현하고 캡슐화 한다.  
persistence : 외부 기술과 연결되는 Layer, DB와의 연결, Security, 외부 API 연결 등이 구현된다.  

  
## DDD와 Layer, ORM
DDD 설계 방식에서는 각 Layer에 대한 경계를 정확하게 지키는 것이 핵심이다.  
해당 경계를 정확하게 지키는 것은 시스템의 유지보수를 쉽게 만들어주고, 확장에 열려있게 만들어준다. 특히 객체지향적인 특성을 살릴 수 있다.  
다만 실제 서비스에서 DDD 설계 구조를 완벽하게 지키기에는 성능적인 이슈가 존재한다.  
다음 상황을 상상해보자.

DDD 설계 패턴에서의 Domain Layer는 DB 기술에 의존해선 안된다. DB 기술 연결은 오롯이 Infrastructure Layer의 담당이기 때문이다. 
따라서 대개 Domain Layer에서는 문제의 영역인 model을 정의 하고, Infrastructure 에서 DB와 연결되는 Entity를 정의, 구현한다.  

특정 데이터를 DB에서 읽은 다음 Client에게 던져주기 위해서는 Entity를 Domain으로 변환하는 Mapping 로직이 필요하다.
이때 DDD 설계패턴의 문제가 발생한다. 게시판이 존재하고, 게시판에는 여러 게시글이 존재하고, 게시글에는 여러 첨부파일이 존재한다고 가정하자.
게시글과 첨부파일은 one to many 관계 이므로, 연관관계 편의 메서드로 연결 되어 있을 것이다. 기본적인 CRUD 구현 방식 에서는 Domain Model = Entity
와 같이 구현하게된다. 따라서 JPA의 Lazy Loading을 사용해서 게시글을 직접 조회하기 전까지는 첨부파일을 DB에서 불러오는 쿼리를 날리지 않도록 구현하는 것이 일반적이다.

하지만, DDD에서는 Entity에서 Lazy Loading을 사용해 첨부파일을 가져오지 않아도, 이를 Model로 Mapping 시키기 때문에 게시판 목록을 조회하려고 해도 전체 게시글을 전부 조회해서 
해당 게시글에 속한 첨부파일을 가져오는 불상사가 발생한다. 즉, 만약 게시판에 글이 10개가 존재한다면, 게시판 목록을 보는 쿼리를 날려도 처음 한번 + 10번의 쿼리가
추가적으로 날아가는 상황이 발생한다. Lazy Loading이 정상적으로 작동하지 않는 것이다.

이러한 상황은 repository의 함수를 두개 만들고, Mapping 함수도 두개 만들어서 처리할 수 있다. 특정 게시글의 정보를 가져오는 
서비스에서는 모든 첨부파일을 가져와야하니 Entity와 Model의 연관관계 편의 메소드를 Mapping해준다.
하지만 게시판 목록을 보여주는 서비스는 굳이 첨부파일을 가져올 필요가 없으니 Entity와 Model 사이의 연관관계 편의 메소드를 연결하지 않는 방식으로 
처리가 가능하다.

## Session Login 과 JWT
모두가 알고있듯이 Http 통신은 stateless하다. 따라서 페이지를 이동할 때마다 해당 사용자가 로그인을 했는지에 대해 인증, 인가의 과정이 필요히다.
여기에서 가장 유명한 방식으로 Session Login과 JWT Login 방식이 존재한다.

### Session Login
세션로그인 방식의 동작과정은 다음과 같다  
1. 사용자가 로그인을 시도한다.
2. 서버에서는 사용자를 검증하고 올바르다면 세션을 생성한다.
3. 서버에서 세션ID를 발급하고, 이를 쿠키에 담아 사용자에게 전달한다.
4. 사용자는 이후에 어떤 요청을 할 때 쿠키를 같이 보낸다.
5. 서버에서는 해당 쿠키에 담긴 세션ID를 확인하고, 세션ID가 유효하다면 요청을 처리한다.

### JWT Login
