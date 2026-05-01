---
name: mcp-builder
description: "MCP(Model Context Protocol) 서버를 구축하는 가이드. 외부 API나 서비스를 LLM과 연동하는 MCP 서버를 만들 때 사용한다. Python(FastMCP) 또는 Node/TypeScript(MCP SDK) 모두 지원한다."
license: 전체 약관은 LICENSE.txt 참조
---

# MCP 서버 개발 가이드

MCP(Model Context Protocol)는 LLM이 잘 설계된 툴을 통해 외부 서비스와 상호작용할 수 있게 한다. MCP 서버의 품질은 LLM이 실제 작업을 얼마나 효과적으로 수행할 수 있는지로 측정한다.

---

# 프로세스

## 전체 워크플로우

고품질 MCP 서버를 만드는 4단계:

### 1단계: 심층 조사 및 계획

#### 1.1 최신 MCP 설계 이해

**API 커버리지 vs. 워크플로우 툴:**
포괄적인 API 엔드포인트 커버리지와 특화된 워크플로우 툴 사이의 균형을 잡는다. 워크플로우 툴은 특정 작업에 편리하고, 포괄적 커버리지는 에이전트가 작업을 조합할 유연성을 준다. 클라이언트마다 성능이 다르다. 불확실하면 포괄적인 API 커버리지를 우선한다.

**툴 이름과 발견 가능성:**
명확하고 설명적인 툴 이름은 에이전트가 올바른 툴을 빠르게 찾는 데 도움이 된다. 일관된 접두사를 사용한다 (예: `github_create_issue`, `github_list_repos`). 동작 중심의 이름을 쓴다.

**컨텍스트 관리:**
에이전트는 간결한 툴 설명과 결과를 필터링/페이지네이션하는 기능에서 이점을 얻는다. 집중된 관련 데이터를 반환하도록 툴을 설계한다.

**실행 가능한 에러 메시지:**
에러 메시지는 구체적인 제안과 다음 단계로 에이전트를 안내해야 한다.

#### 1.2 MCP 프로토콜 문서 학습

사이트맵에서 시작: `https://modelcontextprotocol.io/sitemap.xml`

특정 페이지는 `.md` 접미사로 마크다운 형식으로 가져온다 (예: `https://modelcontextprotocol.io/specification/draft.md`).

검토할 핵심 내용:
- 스펙 개요 및 아키텍처
- 전송 메커니즘 (streamable HTTP, stdio)
- 툴, 리소스, 프롬프트 정의

#### 1.3 프레임워크 문서 학습

**권장 스택:**
- **언어**: TypeScript (SDK 지원 우수, 넓은 호환성. 정적 타이핑과 린팅 도구 지원이 좋아 AI 모델이 TypeScript 코드를 잘 생성함)
- **전송**: 원격 서버는 Streamable HTTP (상태 비저장 JSON, 확장과 유지보수 용이). 로컬 서버는 stdio.

**프레임워크 문서 로드:**

- **MCP 모범 사례**: [📋 모범 사례 보기](./reference/mcp_best_practices.md) - 핵심 가이드라인

**TypeScript (권장):**
- **TypeScript SDK**: WebFetch로 `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/main/README.md` 로드
- [⚡ TypeScript 가이드](./reference/node_mcp_server.md) - TypeScript 패턴과 예제

**Python:**
- **Python SDK**: WebFetch로 `https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md` 로드
- [🐍 Python 가이드](./reference/python_mcp_server.md) - Python 패턴과 예제

#### 1.4 구현 계획

**API 이해:**
서비스의 API 문서를 검토해 핵심 엔드포인트, 인증 요구사항, 데이터 모델을 파악한다. 필요시 웹 검색과 WebFetch를 활용한다.

**툴 선택:**
포괄적인 API 커버리지를 우선한다. 구현할 엔드포인트를 가장 일반적인 작업부터 목록화한다.

---

### 2단계: 구현

#### 2.1 프로젝트 구조 설정

언어별 설정 가이드:
- [⚡ TypeScript 가이드](./reference/node_mcp_server.md) - 프로젝트 구조, package.json, tsconfig.json
- [🐍 Python 가이드](./reference/python_mcp_server.md) - 모듈 구성, 의존성

#### 2.2 핵심 인프라 구현

공유 유틸리티 생성:
- 인증이 포함된 API 클라이언트
- 에러 처리 헬퍼
- 응답 포맷팅 (JSON/Markdown)
- 페이지네이션 지원

#### 2.3 툴 구현

각 툴에 대해:

**입력 스키마:**
- TypeScript는 Zod, Python은 Pydantic 사용
- 제약 조건과 명확한 설명 포함
- 필드 설명에 예제 추가

**출력 스키마:**
- 구조화된 데이터에 가능하면 `outputSchema` 정의
- 툴 응답에 `structuredContent` 사용 (TypeScript SDK 기능)
- 클라이언트가 툴 출력을 이해하고 처리하는 데 도움

**툴 설명:**
- 기능의 간결한 요약
- 파라미터 설명
- 반환 타입 스키마

**구현:**
- I/O 작업에 async/await 사용
- 실행 가능한 메시지로 적절한 에러 처리
- 해당하는 경우 페이지네이션 지원
- 최신 SDK 사용 시 텍스트 콘텐츠와 구조화된 데이터 모두 반환

**어노테이션:**
- `readOnlyHint`: true/false
- `destructiveHint`: true/false
- `idempotentHint`: true/false
- `openWorldHint`: true/false

---

### 3단계: 리뷰 및 테스트

#### 3.1 코드 품질

검토 항목:
- 코드 중복 없음 (DRY 원칙)
- 일관된 에러 처리
- 완전한 타입 커버리지
- 명확한 툴 설명

#### 3.2 빌드 및 테스트

**TypeScript:**
```bash
npm run build
npx @modelcontextprotocol/inspector
```

**Python:**
```bash
python -m py_compile your_server.py
npx @modelcontextprotocol/inspector
```

언어별 상세 테스트 방법과 품질 체크리스트는 가이드 참조.

---

### 4단계: 평가 생성

MCP 서버 구현 후 효과를 검증하는 평가를 만든다.

**[✅ 평가 가이드](./reference/evaluation.md)를 로드해서 완전한 평가 가이드라인을 확인한다.**

#### 4.1 평가 목적 이해

LLM이 MCP 서버를 활용해 현실적이고 복잡한 질문에 효과적으로 답할 수 있는지 테스트한다.

#### 4.2 평가 질문 10개 생성

효과적인 평가를 만드는 과정:

1. **툴 검사**: 사용 가능한 툴 목록과 기능 파악
2. **콘텐츠 탐색**: 읽기 전용 작업으로 데이터 탐색
3. **질문 생성**: 복잡하고 현실적인 질문 10개 작성
4. **답변 검증**: 직접 각 질문을 풀어 답변 확인

#### 4.3 평가 요구사항

각 질문은 다음을 충족해야 한다:
- **독립적**: 다른 질문에 의존하지 않음
- **읽기 전용**: 비파괴적 작업만 필요
- **복잡함**: 여러 툴 호출과 심층 탐색 필요
- **현실적**: 실제 사람이 관심 가질 사용 사례 기반
- **검증 가능**: 문자열 비교로 확인 가능한 단일하고 명확한 답변
- **안정적**: 시간이 지나도 답변이 변하지 않음

#### 4.4 출력 형식

다음 구조의 XML 파일 생성:

```xml
<evaluation>
  <qa_pair>
    <question>Find discussions about AI model launches with animal codenames. One model needed a specific safety designation that uses the format ASL-X. What number X was being determined for the model named after a spotted wild cat?</question>
    <answer>3</answer>
  </qa_pair>
  <!-- 더 많은 qa_pair... -->
</evaluation>
```

---

# 참고 파일

## 문서 라이브러리

개발 중 필요에 따라 로드한다:

### 핵심 MCP 문서 (먼저 로드)
- **MCP 프로토콜**: `https://modelcontextprotocol.io/sitemap.xml` 사이트맵에서 시작, 특정 페이지는 `.md` 접미사로 가져옴
- [📋 MCP 모범 사례](./reference/mcp_best_practices.md) - 다음 포함:
  - 서버 및 툴 이름 규칙
  - 응답 형식 가이드라인 (JSON vs Markdown)
  - 페이지네이션 모범 사례
  - 전송 선택 (streamable HTTP vs stdio)
  - 보안 및 에러 처리 표준

### SDK 문서 (1/2단계에서 로드)
- **Python SDK**: `https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md`에서 가져오기
- **TypeScript SDK**: `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/main/README.md`에서 가져오기

### 언어별 구현 가이드 (2단계에서 로드)
- [🐍 Python 구현 가이드](./reference/python_mcp_server.md) - 다음 포함:
  - 서버 초기화 패턴
  - Pydantic 모델 예제
  - `@mcp.tool`로 툴 등록
  - 완전한 동작 예제
  - 품질 체크리스트

- [⚡ TypeScript 구현 가이드](./reference/node_mcp_server.md) - 다음 포함:
  - 프로젝트 구조 (package.json, tsconfig.json)
  - Zod 스키마 패턴
  - `server.registerTool`로 툴 등록
  - 완전한 동작 예제
  - 품질 체크리스트

### 평가 가이드 (4단계에서 로드)
- [✅ 평가 가이드](./reference/evaluation.md) - 다음 포함:
  - 질문 작성 가이드라인
  - 답변 검증 전략
  - XML 형식 명세
  - 예제 질문과 답변
