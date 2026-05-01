---
name: pr-review-toolkit
description: "PR(풀 리퀘스트) 코드 리뷰를 6개의 전문 에이전트로 수행한다. 코드 작성 후, PR 생성 전, 또는 코드 품질 개선이 필요할 때 사용한다. 에이전트 이름을 직접 언급하거나 ('코드 리뷰해줘', '테스트 커버리지 확인해줘', '에러 처리 점검해줘', '타입 설계 리뷰해줘', '코드 단순화해줘', '주석 검토해줘') 포괄적인 PR 리뷰를 요청할 때 트리거된다."
---

# PR 리뷰 툴킷

6개의 전문 에이전트로 PR의 서로 다른 측면을 분석한다. 요청에 따라 적절한 에이전트 하나 또는 여러 개를 선택해 실행한다.

---

## 에이전트 목록

| 에이전트 | 담당 영역 | 파일 |
|---------|---------|------|
| `code-reviewer` | 프로젝트 가이드라인 준수, 버그 탐지, 코드 품질 | [./agents/code-reviewer.md](./agents/code-reviewer.md) |
| `silent-failure-hunter` | 에러 핸들링 누락, 조용한 실패, 로깅 품질 | [./agents/silent-failure-hunter.md](./agents/silent-failure-hunter.md) |
| `pr-test-analyzer` | 테스트 커버리지 품질, 엣지 케이스, 누락된 테스트 | [./agents/pr-test-analyzer.md](./agents/pr-test-analyzer.md) |
| `comment-analyzer` | 주석 정확성, 문서 품질, 기술 부채 | [./agents/comment-analyzer.md](./agents/comment-analyzer.md) |
| `type-design-analyzer` | 타입 설계 품질, 불변성, 캡슐화 | [./agents/type-design-analyzer.md](./agents/type-design-analyzer.md) |
| `code-simplifier` | 코드 단순화, 복잡도 감소, 가독성 향상 | [./agents/code-simplifier.md](./agents/code-simplifier.md) |

---

## 에이전트 선택 기준

요청 내용을 보고 다음 기준으로 에이전트를 선택한다:

- **"코드 리뷰해줘", "PR 점검해줘"** → `code-reviewer` (기본 리뷰)
- **"에러 처리", "예외 핸들링", "조용한 실패"** → `silent-failure-hunter`
- **"테스트", "커버리지", "테스트 빠진 거"** → `pr-test-analyzer`
- **"주석", "문서", "docstring"** → `comment-analyzer`
- **"타입", "인터페이스", "타입 설계"** → `type-design-analyzer`
- **"단순화", "리팩토링", "복잡한 코드"** → `code-simplifier`
- **"전체 리뷰", "종합 리뷰", "PR 올리기 전에"** → 여러 에이전트 조합

여러 에이전트가 필요한 경우 독립적인 에이전트들을 병렬로 실행한다.

---

## 권장 워크플로우

### 코드 작성 후
1. `code-reviewer` — 가이드라인 준수 및 버그 확인

### 에러 핸들링 추가 후
2. `silent-failure-hunter` — 조용한 실패 점검

### 테스트 작성 후
3. `pr-test-analyzer` — 커버리지 확인

### 문서화 후
4. `comment-analyzer` — 주석 정확성 검증

### PR 최종 확인
5. `code-simplifier` — 코드 정리 마무리

---

## 실행 방법

1. 요청에 맞는 에이전트 파일을 Read 툴로 읽는다
2. 에이전트 파일의 지시에 따라 리뷰를 수행한다
3. 기본 리뷰 범위: `git diff`로 확인한 변경 사항 (다른 범위를 지정하면 그에 따름)

포괄적인 리뷰를 요청받은 경우 `code-reviewer`를 먼저 실행하고, 결과에 따라 추가 에이전트를 선택한다.
