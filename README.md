# Spring-Api-Response-Template

Spring Boot 3.x.x 버전 & Java 17+ 환경에서 작동합니다.<br>
백엔드 API 개발 시 클라이언트(프론트엔드/앱)와 일관된 통신을 하기 위한 공통 응답 및 전역 예외 처리 템플릿입니다.

# Issue

* OCP 준수 에러 처리: 모든 에러를 하나의 거대한 Enum에서 관리할 때 발생하는 병목 현상을 해결하기 위해, `ErrorCode`를 인터페이스(Interface)로 선언하였습니다. 이를 통해 도메인별(`UserErrorCode`, `BoardErrorCode` 등)로 에러 코드를 분리 및 확장할 수 있습니다.
* 응답 포맷 캡슐화: `ResponseDto` 객체 생성 시 무분별한 `new` 키워드 사용을 막고 `ofSuccess()`, `ofFailure()` 등의 정적 팩토리 메서드만 사용하도록 제한하였습니다.

# Manual

이 코드는 `com.practice.apiresponse.global` 패키지 하위에 위치하며 크게 `dto`와 `exception` 두 가지 역할을 담당합니다.

**1. 정상 응답 처리 (Success)**

성공 시에는 제네릭을 활용하여 다양한 형태의 데이터를 일관된 포맷으로 감싸서 반환합니다. 데이터가 필요 없는 경우(단순 삭제 등)에는 `Void`를 사용합니다.

**2. 도메인별 에러 코드 정의**

`ErrorCode` 인터페이스를 상속받아 각 도메인에 맞는 에러를 Enum으로 정의합니다.

```java
@Getter
@RequiredArgsConstructor
public enum UserErrorCode implements ErrorCode {
    USER_NOT_FOUND(HttpStatus.NOT_FOUND, "U001", "사용자를 찾을 수 없습니다.");

    private final HttpStatus httpStatus;
    private final String errorCode;
    private final String message;
}
```
**3. 비즈니스 예외 발생**

서비스 로직에서 예외 상황이 발생하면 `ApiException`을 던집니다. 이후 처리는 `@RestControllerAdvice`가 담당합니다.

```Java
if (user == null) {
    throw new ApiException(UserErrorCode.USER_NOT_FOUND);
}
```

**4. 컨트롤러 적용 예시**

```Java
@GetMapping("/{id}")
public ResponseEntity<ResponseDto<UserDto>> getUser(@PathVariable Long id) {
    UserDto userDto = userService.getUser(id);
    // 상태 코드 200(OK)과 함께 일관된 포맷 반환
    return ResponseEntity.ok(ResponseDto.ofSuccess(SuccessMessage.OPERATION_SUCCESS, userDto));
}
```

# Caution
이 코드는 백엔드 공통 뼈대 코드로써 동작합니다. 따라서, 전체 과정을 테스트하거나 활용하기 위해서는 실제 비즈니스 도메인(Controller, Service) 코드를 작성하여 연결해야 합니다.

응답 포맷 파싱: 프론트엔드 연동 시, HTTP 상태 코드(200, 400, 500 등)는 헤더(Header)를 통해 전달되며, 바디(Body)에는 아래와 같은 커스텀 JSON 포맷이 전달됩니다. 프론트엔드에서는 이 구조를 기준으로 파싱 로직을 작성해야 합니다.

```JSON
{
  "code": "SUCCESS", 
  "message": "요청이 성공적으로 처리되었습니다.",
  "content": { ... 실제 데이터 ... } 
}
```

에러 발생 시 `content`는 `null`로 반환되며, `code`에 커스텀 에러 코드(예: "U001")가 담깁니다.

# 📂 패키지 구조 (Directory Structure)

```text
com.practice.apiresponse.global
 ├── dto
 │    ├── ResponseDto.java       // 공통 응답 레코드 (code, message, content)
 │    └── SuccessMessage.java    // 성공 상태 메시지를 관리하는 Enum
 └── exception
      ├── ApiException.java      // 비즈니스 로직 처리를 위한 커스텀 예외
      ├── ApiExceptionHandler.java // 전역 예외 처리기
      └── ErrorCode.java         // 도메인별 에러 코드 확장을 위한 인터페이스
