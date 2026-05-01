---
name: webapp-testing
description: "Playwright를 사용해 로컬 웹 애플리케이션을 테스트하고 상호작용하는 툴킷. 프론트엔드 기능 검증, UI 동작 디버깅, 브라우저 스크린샷 캡처, 콘솔 로그 확인 시 사용한다. '테스트해줘', '브라우저에서 확인해줘', '버튼 클릭 동작 확인', 'UI 자동화' 등의 요청에 트리거된다."
license: 전체 약관은 LICENSE.txt 참조
---

# 웹 애플리케이션 테스트

로컬 웹 앱을 테스트하려면 Python Playwright 스크립트를 직접 작성해 실행한다.

**설치 위치**: `~/.config/opencode/skills/webapp-testing/`

**헬퍼 스크립트**:
- `scripts/with_server.py` — 서버 수명주기 관리 (여러 서버 동시 지원)

**스크립트는 먼저 `--help`로 사용법을 확인한다.** 소스 코드를 직접 읽지 않는다 — 스크립트가 크기 때문에 컨텍스트를 낭비한다. 블랙박스로 호출하면 된다.

---

## 접근법 결정 트리

```
작업 유형 파악
    ├─ 정적 HTML 파일인가?
    │   ├─ Yes → HTML 파일을 직접 읽어 셀렉터 파악
    │   │         ├─ 성공 → 셀렉터로 Playwright 스크립트 작성
    │   │         └─ 실패/불완전 → 아래 동적 웹앱으로 처리
    │   │
    │   └─ No (동적 웹앱)
    │         ├─ 서버가 이미 실행 중?
    │         │   ├─ No → with_server.py로 서버 시작 후 자동화 스크립트 실행
    │         │   └─ Yes → 정찰 후 행동 패턴 (아래 참조)
```

---

## with_server.py 사용법

먼저 `--help`로 사용법 확인:

```bash
python ~/.config/opencode/skills/webapp-testing/scripts/with_server.py --help
```

**단일 서버:**
```bash
python ~/.config/opencode/skills/webapp-testing/scripts/with_server.py \
  --server "npm run dev" --port 5173 \
  -- python your_automation.py
```

**다중 서버 (백엔드 + 프론트엔드):**
```bash
python ~/.config/opencode/skills/webapp-testing/scripts/with_server.py \
  --server "cd backend && python server.py" --port 3000 \
  --server "cd frontend && npm run dev" --port 5173 \
  -- python your_automation.py
```

자동화 스크립트에는 Playwright 로직만 작성한다 (서버 관리는 with_server.py가 처리):
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)  # 항상 headless 모드
    page = browser.new_page()
    page.goto('http://localhost:5173')
    page.wait_for_load_state('networkidle')  # 중요: JS 실행 완료까지 대기
    # ... 자동화 로직
    browser.close()
```

---

## 정찰 후 행동 패턴

동적 웹앱의 경우, 직접 셀렉터를 추측하지 말고 먼저 현재 상태를 파악한다.

**1단계: 렌더링된 DOM 검사**
```python
page.wait_for_load_state('networkidle')  # 반드시 먼저 대기
page.screenshot(path='/tmp/inspect.png', full_page=True)
content = page.content()
buttons = page.locator('button').all()
```

**2단계: 검사 결과에서 셀렉터 파악**

**3단계: 발견한 셀렉터로 액션 실행**

---

## 자주 하는 실수

❌ `networkidle` 대기 전에 동적 앱의 DOM 검사  
✅ 검사 전 반드시 `page.wait_for_load_state('networkidle')` 실행

---

## 모범 사례

- **스크립트는 블랙박스로 사용**: `scripts/`의 스크립트는 소스를 읽지 말고 `--help`로 사용법 확인 후 직접 호출
- 동기 스크립트에는 `sync_playwright()` 사용
- 브라우저는 항상 `headless=True`로 실행
- 완료 후 반드시 브라우저 닫기
- 명확한 셀렉터 사용: `text=`, `role=`, CSS 셀렉터, ID
- 적절한 대기 추가: `page.wait_for_selector()` 또는 `page.wait_for_timeout()`

---

## 예제 파일

`~/.config/opencode/skills/webapp-testing/examples/`에서 패턴 참조:

| 파일 | 내용 |
|------|------|
| `element_discovery.py` | 버튼, 링크, 입력 필드 탐색 |
| `static_html_automation.py` | 로컬 HTML 파일 자동화 (`file://` URL) |
| `console_logging.py` | 브라우저 콘솔 로그 캡처 |

예제를 참고할 때는 Read 툴로 해당 파일을 읽는다.

---

## 사전 요구사항

```bash
pip install playwright
playwright install chromium
```
