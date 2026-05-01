# MCP 서버 모범 사례

## 빠른 참조

### 서버 이름
- **Python**: `{service}_mcp` (예: `slack_mcp`)
- **Node/TypeScript**: `{service}-mcp-server` (예: `slack-mcp-server`)

### 툴 이름
- snake_case에 서비스 접두사 포함
- 형식: `{service}_{action}_{resource}`
- 예: `slack_send_message`, `github_create_issue`

### 응답 형식
- JSON과 Markdown 형식 모두 지원
- JSON: 프로그래밍 방식 처리용
- Markdown: 사람이 읽기 편한 형식

### 페이지네이션
- 항상 `limit` 파라미터 준수
- `has_more`, `next_offset`, `total_count` 반환
- 기본값은 20~50개

### 전송 방식
- **Streamable HTTP**: 원격 서버, 다중 클라이언트
- **stdio**: 로컬 통합, 커맨드라인 툴
- SSE는 사용하지 않음 (streamable HTTP로 대체됨)

---

## 서버 이름 규칙

표준화된 이름 패턴을 따른다:

**Python**: `{service}_mcp` 형식 (소문자, 언더스코어)
- 예: `slack_mcp`, `github_mcp`, `jira_mcp`

**Node/TypeScript**: `{service}-mcp-server` 형식 (소문자, 하이픈)
- 예: `slack-mcp-server`, `github-mcp-server`, `jira-mcp-server`

이름은 일반적이고, 서비스를 설명하며, 태스크 설명에서 유추 가능하고, 버전 번호 없이 작성한다.

---

## 툴 이름 및 설계

### 툴 이름

1. **snake_case 사용**: `search_users`, `create_project`, `get_channel_info`
2. **서비스 접두사 포함**: 여러 MCP 서버와 함께 사용될 것을 고려
   - `send_message` 대신 `slack_send_message`
   - `create_issue` 대신 `github_create_issue`
3. **동작 중심**: 동사로 시작 (get, list, search, create 등)
4. **구체적으로**: 다른 서버와 충돌할 수 있는 일반적인 이름 피하기

### 툴 설계

- 툴 설명은 기능을 좁고 명확하게 설명
- 설명은 실제 기능과 정확히 일치
- 툴 어노테이션 제공 (readOnlyHint, destructiveHint, idempotentHint, openWorldHint)
- 툴 동작은 집중적이고 원자적으로 유지

---

## 응답 형식

데이터를 반환하는 모든 툴은 여러 형식을 지원해야 한다:

### JSON 형식 (`response_format="json"`)
- 기계가 읽을 수 있는 구조화된 데이터
- 모든 필드와 메타데이터 포함
- 일관된 필드 이름과 타입
- 프로그래밍 방식 처리에 사용

### Markdown 형식 (`response_format="markdown"`, 보통 기본값)
- 사람이 읽기 편한 형식
- 명확성을 위해 헤더, 목록, 포맷팅 사용
- 타임스탬프를 사람이 읽을 수 있는 형식으로 변환
- 괄호 안에 ID와 함께 표시 이름 표시
- 장황한 메타데이터 생략

---

## 페이지네이션

리소스를 나열하는 툴의 경우:

- **항상 `limit` 파라미터를 준수**
- **페이지네이션 구현**: `offset` 또는 커서 기반 페이지네이션 사용
- **페이지네이션 메타데이터 반환**: `has_more`, `next_offset`/`next_cursor`, `total_count` 포함
- **전체 결과를 메모리에 로드하지 않기**: 대용량 데이터셋에서 특히 중요
- **합리적인 기본 한도**: 일반적으로 20~50개

페이지네이션 응답 예:
```json
{
  "total": 150,
  "count": 20,
  "offset": 0,
  "items": [...],
  "has_more": true,
  "next_offset": 20
}
```

---

## 전송 옵션

### Streamable HTTP

**적합한 경우**: 원격 서버, 웹 서비스, 다중 클라이언트

**특징**:
- HTTP를 통한 양방향 통신
- 여러 클라이언트 동시 지원
- 웹 서비스로 배포 가능
- 서버에서 클라이언트로 알림 지원

**사용 시점**:
- 여러 클라이언트를 동시에 서비스할 때
- 클라우드 서비스로 배포할 때
- 웹 애플리케이션과 통합할 때

### stdio

**적합한 경우**: 로컬 통합, 커맨드라인 툴

**특징**:
- 표준 입출력 스트림 통신
- 간단한 설정, 네트워크 구성 불필요
- 클라이언트의 서브프로세스로 실행

**사용 시점**:
- 로컬 개발 환경용 툴을 만들 때
- 데스크톱 애플리케이션과 통합할 때
- 단일 사용자, 단일 세션 시나리오

**주의**: stdio 서버는 stdout에 로그를 출력하면 안 됨 (stderr 사용)

### 전송 선택 기준

| 기준 | stdio | Streamable HTTP |
|------|-------|-----------------|
| **배포** | 로컬 | 원격 |
| **클라이언트** | 단일 | 다중 |
| **복잡도** | 낮음 | 중간 |
| **실시간** | 아니요 | 예 |

---

## 보안 모범 사례

### 인증 및 권한

**OAuth 2.1**:
- 인증된 기관의 인증서로 보안 OAuth 2.1 사용
- 요청 처리 전 액세스 토큰 검증
- 서버를 위해 특별히 발급된 토큰만 수락

**API 키**:
- API 키는 환경 변수에 저장, 코드에 절대 포함하지 않음
- 서버 시작 시 키 검증
- 인증 실패 시 명확한 에러 메시지 제공

### 입력 검증

- 디렉토리 순회를 막기 위해 파일 경로 검사
- URL 및 외부 식별자 검증
- 파라미터 크기와 범위 확인
- 시스템 호출에서 명령 주입 방지
- 모든 입력에 스키마 검증 (Pydantic/Zod) 사용

### 에러 처리

- 내부 에러를 클라이언트에 노출하지 않음
- 보안 관련 에러는 서버 측에 로그
- 도움이 되지만 과도한 정보는 담지 않는 에러 메시지
- 에러 발생 후 리소스 정리

### DNS 리바인딩 방어

로컬에서 실행되는 streamable HTTP 서버의 경우:
- DNS 리바인딩 방어 활성화
- 모든 수신 연결에서 `Origin` 헤더 검증
- `0.0.0.0` 대신 `127.0.0.1`에 바인딩

---

## 툴 어노테이션

클라이언트가 툴 동작을 이해하도록 어노테이션을 제공한다:

| 어노테이션 | 타입 | 기본값 | 설명 |
|-----------|------|--------|------|
| `readOnlyHint` | boolean | false | 툴이 환경을 수정하지 않음 |
| `destructiveHint` | boolean | true | 툴이 파괴적인 업데이트를 수행할 수 있음 |
| `idempotentHint` | boolean | false | 동일한 인수로 반복 호출해도 추가 효과 없음 |
| `openWorldHint` | boolean | true | 툴이 외부 엔티티와 상호작용함 |

**중요**: 어노테이션은 힌트이지 보안 보장이 아님. 클라이언트는 어노테이션만을 기반으로 보안 결정을 내리면 안 됨.

---

## 에러 처리

- 표준 JSON-RPC 에러 코드 사용
- 툴 에러는 결과 객체 내에서 보고 (프로토콜 수준 에러가 아님)
- 다음 단계를 제안하는 도움이 되고 구체적인 에러 메시지 제공
- 내부 구현 세부사항 노출 금지
- 에러 발생 시 리소스를 적절히 정리

에러 처리 예:
```typescript
try {
  const result = performOperation();
  return { content: [{ type: "text", text: result }] };
} catch (error) {
  return {
    isError: true,
    content: [{
      type: "text",
      text: `Error: ${error.message}. Try using filter='active_only' to reduce results.`
    }]
  };
}
```

---

## 테스트 요구사항

포괄적인 테스트가 다음을 커버해야 한다:

- **기능 테스트**: 유효/무효 입력으로 올바른 실행 확인
- **통합 테스트**: 외부 시스템과의 상호작용 테스트
- **보안 테스트**: 인증, 입력 검사, 속도 제한 검증
- **성능 테스트**: 부하 및 타임아웃 동작 확인
- **에러 처리**: 적절한 에러 보고 및 정리 확인

---

## 문서화 요구사항

- 모든 툴과 기능에 대한 명확한 문서 제공
- 작동하는 예제 포함 (주요 기능당 최소 3개)
- 보안 고려사항 문서화
- 필요한 권한 및 접근 수준 명시
- 속도 제한 및 성능 특성 문서화
