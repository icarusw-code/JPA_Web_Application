[TOC]



# API 개발과 성능 최적화

## API 개발 기본

### 회원 등록 API

 **V1 - 엔티티를 Request Body에 직접 매핑**

```java
@RestController
@RequiredArgsConstructor
public class MemberApiController{
	private final MemberService memberService;
    
    @PostMapping("/api/v1/members")
    public CreateMemberResponse saveMemberV1(@RequestBody @Valid Member member){
        Long id = memberService.join(member);
        return new CreateMemberResponse(id);
    }
    
    @Data
    static class CreateMemberRequest {
        private String name; 
    }
    
    @Data
    static class CreateMemberResponse {
        private Long id;
        public CreateMemberResponse(Long id) {
            this.id = id; 
        }
    }
}
```

**문제점**

- 엔티티에 프레젠테이션 계층을 위한 로직이 추가된다.
- 엔티티에 API 검증을 위한 로직이 들어간다(@NotEmpty 등등)
- 회원 엔티티를 위한 다양한 API가 만들어지는데, 한 엔티티에 각각의 API를 위한 모든 요청 요구사항을 담기 어렵다.
- **엔티티가 변경되면 API 스펙이 변한다.**

결론 

- API요청 스펙에 맞추어 별도의 DTO를 파라미터로 받도록 한다.



**V2 - 엔티티 대신에 DTO를 RequestBody에 매핑**

```java
@PostMapping("/api/v2/members")
public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request){
    
        Member member = new Member();         
        member.setName(request.getName());
    
        Long id = memberService.join(member);
        return new CreateMemberResponse(id); 
}

@Data
static class CreateMemberRequest{
    private String name;
}
```

- CreateMemberRequest를 Member 엔티티 대신에 RequestBody와 매핑한다.
- 엔티티와 프레젠테이션 계층을 위한 로직을 분리할 수 있다.
- 엔티티와 API 스펙을 명확하게 분리할 수 있다.



### 회원 수정 API

```java
@PutMapping("/api/v2/members/{id}")
public UpdateMemberResponse updateMemberV2(@PathVariable("id") Long id, @RequestBody @Valid UpdateMemberRequest request) 
{
    memberService.update(id, request.getName());
    Member findMember = memberService.findOne(id);
    return new UpdateMemberResponse(findMember.getId(), findMember.getName());
}

@Data
static class UpdateMemberRequest{
    private String name;
}

@Data
@AllArgsConstructor
static class UpdateMemberResponse {
    private Long id; 
    private String name;
}
```

- 회원 수정도 DTO를 요청 파라미터에 매핑한다.

```java
public class MemberService {

    private final MemberRepository memberRepository;
    
        @Transactional
        public void update(Long id, String name) {
            Member member = memberRepository.findOne(id);
            member.setName(name);
    }
}
```

- 변경 감지를 사용해서 데이터를 수정한다.



### 회원 조회 API

**V1 - 응답 값으로 엔티티를 직접 외부에 노출**

```java
package jpabook.jpashop.api;

@RestController
@RequiredArgsConstructor
public class MemberApiController {
    
    private final MemberService memberService;
    
    @GetMapping("/api/v1/members")
    public List<Member> membersV1() {
        return memberService.findMembers(); 
    }
}
```

**문제점**

- 기본적으로 엔티티의 모든 값이 노출된다.
- 응답 스펙을 맞추기 위해 로직이 추가된다.(@JsonIgnore, 별도의 뷰 로직 등등)
- 엔티티가 변경되면 API 스펙이 변한다.
- 추가로 컬렉션을 직접 반환하면 향후 API 스펙을 변경하기 어렵다.

**결론**

- 어떤 API는 name 필드가 필요하지만, 어떤 API는 name필드가 필요없을 수 있다. 

  ---> API 응답 스펙에 맞추어 별도의 DTO를 반환한다.



**V2 - 응답 값으로 엔티티가 아닌 별도의 DTO사용**

```java
@GetMapping("/api/v2/members") 
public Result membersV2() {
    
    List<Member> findMembers = memberService.findMembers();
    //엔티티 -> DTO 변환
    List<MemberDto> collect = findMembers.stream()
			.map(m -> new MemberDto(m.getName())) 
        	.collect(Collectors.toList());
    
    return new Result(collect); 
}

@Data
@AllArgsConstructor 
static class Result<T> {
    private T data; 
}
@Data
@AllArgsConstructor 
static class MemberDto {
    private String name; 
}
```

- 엔티티를 DTO로 변환해서 반환한다.
- 엔티티가 변해도 API 스펙이 변경되지 않는다.
- 추가로 Result 클래스로 컬렉션을 감싸서 향후 필요한 필드를 추가할 수 있다.



























