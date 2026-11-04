# openapi.yaml 운영 규칙 (Stage 3 — Contract)

필드별 작성법·YAML 예시·케이스별 패턴은 노션 **"openapi.yaml 작성 가이드"** 에 있다.
이 문서는 **지켜야 하는 규칙**만 담는다. 두 문서가 어긋나면 **이 문서가 기준**이다.

## 기준 문서

- 코드 레포와 계약이 어긋나면 **이 레포의 `openapi.yaml`이 기준**이다. 코드를 고친다.
- 계약은 **spec → openapi → 코드** 순으로 반영한다.
- **spec이 승인된 뒤에만** 작성한다. spec보다 먼저 API를 그리지 않는다.
- spec에 없는 필드·엔드포인트를 만들지 않는다.

## 파일 구조

```
openapi.yaml                          # 계약의 진입점.
api/
├── paths/
│   ├── _TEMPLATE.yaml                # 도메인 1개용 path 파일 템플릿
│   └── {도메인}-{동작}.yaml           # 도메인별 operation 실제 정의
└── components/
    ├── schemas/
    │   ├── _TEMPLATE.yaml            # 재사용 스키마 템플릿
    │   └── {이름}.yaml
    ├── responses/
    │   ├── _TEMPLATE.yaml            # 재사용 응답 템플릿
    │   └── {이름}.yaml
    └── parameters/
        ├── _TEMPLATE.yaml            # 재사용 파라미터 템플릿
        └── {이름}.yaml
```

| 경로 | 역할 |
|---|---|
| `openapi.yaml` | 계약 전체의 진입점. 실제 HTTP 경로(`/api/v1/auth/login` 등)를 키로 두고, 그 경로가 참조할 조각을 `$ref`로 끌어온다. |
| `api/paths/{도메인}-{동작}.yaml` | 도메인 하나의 operation들을 모아둔 파일. 파일 안에는 실제 URL을 적지 않는다(그건 root의 역할). |
| `api/components/schemas/*.yaml` | 여러 operation이 공유하는 요청/응답 **바디 모양**. 예: `Success`, `Error`. |
| `api/components/responses/*.yaml` | 여러 operation이 공유하는 **응답 전체**(설명 + 상태 코드 의미 + 바디 스키마 참조). 예: `ValidationError`. |
| `api/components/parameters/*.yaml` | 여러 operation이 공유하는 경로·쿼리 파라미터 정의. (아직 실제 예시 없음, 필요해지면 추가) |

`securitySchemes`(인증 방식 자체의 정의)는 별도 파일로 쪼개지 않고 root `openapi.yaml`의 `components.securitySchemes`에 직접 둔다. operation에서는 이름만 참조한다.

## 명명 규칙

| 대상 | 규칙 |
|---|---|
| 파일명 | `{도메인}-{동작}.yaml`. **spec 파일명과 일치**시킨다 (`spec/USR-AUTH.md` ↔ `api/paths/USR-AUTH.yaml`) |
| `operationId` | **spec 기능 ID 그대로** (`USR-AUTH-001`). 임의로 다르게 붙이면 추적성이 깨진다 |
| operation 키 | 캐멀케이스(`sendVerificationCode`). 파일 안에서만 쓰는 이름이며 URL이 아니다 |
| `tags` | 도메인명 하나로 통일 (`[USR-AUTH]`) |
| 에러 `code` | 대문자 스네이크 (`EMAIL_DUPLICATED`) |

## operation 필수 항목

모든 operation은 `operationId`·`tags`·`summary`·`description`·`security`·`responses`를 빠짐없이 갖는다.

- `security`는 **항상 명시**한다. 인증이 필요 없으면 `security: []`. 상속되는 기본값에 의존하지 않는다.
- `content`의 키는 항상 `application/json`이다. 이 레포의 API는 JSON만 주고받는다.
- 요청 본문이 없는 API는 `requestBody`를 생략하고, `description`의 입력 제약에 "요청 본문 없음"과 필요한 쿠키를 적는다.
- spec에 형식 규칙(길이·패턴)이 있으면 `minLength`/`maxLength`/`pattern`으로 **스키마에 인코딩**한다. `description` 글로만 남기지 않는다.
- `description`에는 spec "처리 로직"의 숫자·정책(TTL, 만료, rotation 등)을 옮겨 적는다. 코드 작성자가 openapi만 보고 규칙을 알 수 있어야 한다.
- 반대로 **스키마를 읽으면 아는 내용은 `description`에 다시 쓰지 않는다.** 필드 설명은 이름만으로 뜻이 안 잡히는 것(`러닝타임(분)`, `시작시간(HH:mm)`, `주최`)에만 붙이고, 널 필드 제외·페이지 없음 같은 공통 동작은 적지 않는다(각각 "공통 응답 계약"과 `Page.yaml`에 있다).
- **필터 쿼리 파라미터에 `ALL` 같은 "전체" 값을 두지 않는다.** enum에는 실제 상태 값만 넣고, 파라미터를 보내지 않는 것이 전체 조회다. `description`에 `선택하지 않으면 전체 조회`라고 적는다.
- 파일 맨 위에는 operation 키 ↔ 실제 경로 매핑 주석표를 남긴다.

## 공통 응답 계약

모든 응답은 `{code, message, data}` 래퍼로 내려간다(백엔드 `ApiResponse<T>`).
**값이 null인 필드는 직렬화에서 제외**되므로, 성공 응답에는 `code`가 없고 실패 응답에는 `data`가 없다.

| 상황 | 실제 바디 | 쓰는 스키마 |
|---|---|---|
| 성공(데이터 없음) | `{"message":"success"}` | `Success.yaml` |
| 성공(데이터 있음) | `{"message":"success","data":{...}}` | `Success.yaml` + `data`를 `allOf`로 조합 |
| 실패 | `{"code":"EMAIL_DUPLICATED","message":"..."}` | `Error.yaml` |

- 성공 `message`는 **항상 `"success"` 고정**이다. 사용자에게 보여줄 문구는 프론트가 각 화면에서 정하므로 **계약에 적지 않는다.** 성공 응답의 `description`은 `공고 생성 완료`·`취소 완료`처럼 무슨 일이 끝났는지만 적는다.
- 반환할 데이터는 **반드시 `data` 안에** 넣는다. 래퍼 정의를 복제하지 않도록 `allOf`로 조합한다.
- 같은 모양의 응답은 `Success`·`Error`·`ValidationError`를 재사용한다. 정말 새로운 모양일 때만 `api/components/`에 추가하고 root `components`에도 등록한다.

## 에러 코드 정책

- **프론트는 `message`가 아니라 `code`로 분기한다.** `message`는 바뀔 수 있는 표시용 텍스트이고, `code`가 계약이다.
- 네이밍:
  - 입력 형식 검증 실패 → 공통 `INVALID_INPUT` 재사용
  - 그 외 비즈니스 규칙 위반 → 도메인 코드를 새로 정의 (`EMAIL_DUPLICATED`, `RESET_TOKEN_INVALID`)
- **코드는 이 계약에서 먼저 정의하고, 백엔드가 그 이름 그대로 도메인 `ErrorCode` enum으로 구현한다.**
- 실제로 반환할 `code`·`message`는 **각 응답의 `example`/`examples`에 실제 값으로** 남긴다. 공통 schema에 박힌 `example`은 대표 예시일 뿐 기본값이 아니다.
- 한 상태 코드에 원인이 여럿이면 `example`(단수) 대신 `examples`(복수)로 나눈다.
- 응답 **전체**를 `$ref`로 재사용하면 호출부에서 `example`을 덮어쓸 수 없다. 그 경우 대표 문구는 재사용되는 응답 파일 자체에 적는다.
- spec에 문구가 없으면 서버가 내려줄 문구를 추론해 채우되, **리뷰어에게 문구 확정이 필요하다고 알린다.** 계약 안에 남기는 경우에는 해당 필드·파라미터의 `description`에 `~ 확정 필요` 한 줄로 붙인다. 긴 설명 문단으로 늘리지 않는다.

## 예외 케이스 → 상태 코드 매핑

spec의 "예외 케이스" 항목은 **빠짐없이 하나씩** 응답으로 매핑한다.

| 상황 | 코드 |
|---|---|
| 입력 형식 오류 | `400` (재사용: `ValidationError`) |
| 인증/권한 없음, 토큰 만료·불일치 | `401` |
| 계정 정지 등 접근 자체를 막는 경우 | `403` |
| 대상 없음(존재하지 않는 리소스, 남의 리소스) | `404` |
| 중복(이미 가입된 이메일 등), 현재 상태로는 할 수 없는 동작 | `409` |
| 외부 서비스 장애(이메일 발송 실패 등) | `502` |

## 작성 후 확인

```
npm run lint     # redocly 스펙 검사 ($ref 깨짐·문법 오류)
npm run swagger  # 번들 후 http://localhost:4000 에서 렌더링 확인
```

`swagger/openapi.yaml`은 `npm run bundle`이 만드는 **생성물**이다(gitignore 대상). 직접 수정하지 않는다.
lint 통과와 Swagger UI 렌더링을 확인한 뒤 커밋한다.

## 새 도메인 추가 체크리스트

- [ ] `api/paths/_TEMPLATE.yaml`을 복사해 `api/paths/{도메인}-{동작}.yaml` 생성
- [ ] 파일 맨 위에 operation 키 ↔ 실제 경로 매핑 주석표 작성
- [ ] spec 기능 하나 = operation 하나로 빠짐없이 매핑
- [ ] root `openapi.yaml`의 `paths:`에 실제 URL → `$ref` 매핑 추가
- [ ] root `openapi.yaml`의 `tags:`에 도메인명과 설명(구현 범위 포함) 등록
- [ ] 새로 만든 공통 스키마·응답이 있으면 root `components`에도 등록
- [ ] `npm run lint` 통과
- [ ] `npm run swagger`로 렌더링 확인

## 하지 말 것

- spec에 없는 필드·엔드포인트를 새로 만들지 않는다.
- **이미 커밋된 `operationId`·실제 경로는 바꾸지 않는다.** 코드 레포가 이 값을 참조한다.
- `api/paths/*.yaml` 안에 실제 URL을 적지 않는다. 경로를 정하는 곳은 root `openapi.yaml`의 `paths:`뿐이다.
- `security`를 생략하지 않는다. 인증이 필요 없으면 `security: []`.
- 생성물인 `swagger/openapi.yaml`을 직접 수정하지 않는다.
- 같은 모양의 응답을 새 스키마로 복제하지 않는다.
