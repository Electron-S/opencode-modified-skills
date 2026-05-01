# Python MCP 서버 구현 가이드

## 개요

MCP Python SDK를 사용해 MCP 서버를 구축하는 Python 전용 모범 사례 및 예제. 서버 설정, 툴 등록 패턴, Pydantic 입력 검증, 에러 처리, 완전한 예제 포함.

---

## 빠른 참조

### 핵심 임포트
```python
from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel, Field, field_validator, ConfigDict
from typing import Optional, List, Dict, Any
from enum import Enum
import httpx
```

### 서버 초기화
```python
mcp = FastMCP("service_mcp")
```

### 툴 등록 패턴
```python
@mcp.tool(name="tool_name", annotations={...})
async def tool_function(params: InputModel) -> str:
    # 구현
    pass
```

---

## MCP Python SDK와 FastMCP

공식 MCP Python SDK가 FastMCP를 제공한다. FastMCP의 기능:
- 함수 시그니처와 docstring에서 description 및 inputSchema 자동 생성
- 입력 검증을 위한 Pydantic 모델 통합
- `@mcp.tool` 데코레이터 기반 툴 등록

**완전한 SDK 문서를 보려면 WebFetch를 사용:**
`https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md`

## 서버 이름 규칙

Python MCP 서버 이름 패턴:
- **형식**: `{service}_mcp` (소문자, 언더스코어)
- **예**: `github_mcp`, `jira_mcp`, `stripe_mcp`

일반적이고, 서비스를 설명하며, 버전 번호 없이 작성한다.

## 툴 구현

### 툴 이름

snake_case 사용, 서비스 컨텍스트 포함:
- `send_message` 대신 `slack_send_message`
- `create_issue` 대신 `github_create_issue`
- `list_tasks` 대신 `asana_list_tasks`

### FastMCP 툴 구조

```python
from pydantic import BaseModel, Field, ConfigDict
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("example_mcp")

class ServiceToolInput(BaseModel):
    '''Input model for service tool operation.'''
    model_config = ConfigDict(
        str_strip_whitespace=True,
        validate_assignment=True,
        extra='forbid'
    )

    param1: str = Field(..., description="First parameter (e.g., 'user123')", min_length=1, max_length=100)
    param2: Optional[int] = Field(default=None, description="Optional integer", ge=0, le=1000)
    tags: Optional[List[str]] = Field(default_factory=list, description="List of tags", max_items=10)

@mcp.tool(
    name="service_tool_name",
    annotations={
        "title": "Human-Readable Tool Title",
        "readOnlyHint": True,
        "destructiveHint": False,
        "idempotentHint": True,
        "openWorldHint": False
    }
)
async def service_tool_name(params: ServiceToolInput) -> str:
    '''Tool description automatically becomes the 'description' field.

    Args:
        params (ServiceToolInput): Validated input parameters containing:
            - param1 (str): First parameter description
            - param2 (Optional[int]): Optional parameter with default

    Returns:
        str: JSON-formatted response
    '''
    pass
```

## Pydantic v2 핵심 기능

- 중첩된 `Config` 클래스 대신 `model_config` 사용
- 더 이상 사용되지 않는 `validator` 대신 `field_validator` 사용
- 더 이상 사용되지 않는 `dict()` 대신 `model_dump()` 사용
- 검증자는 `@classmethod` 데코레이터 필요
- 검증자 메서드에 타입 힌트 필수

```python
from pydantic import BaseModel, Field, field_validator, ConfigDict

class CreateUserInput(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True, validate_assignment=True)

    name: str = Field(..., description="User's full name", min_length=1, max_length=100)
    email: str = Field(..., description="User's email address", pattern=r'^[\w\.-]+@[\w\.-]+\.\w+$')
    age: int = Field(..., description="User's age", ge=0, le=150)

    @field_validator('email')
    @classmethod
    def validate_email(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Email cannot be empty")
        return v.lower()
```

## 응답 형식

```python
from enum import Enum

class ResponseFormat(str, Enum):
    MARKDOWN = "markdown"
    JSON = "json"

class UserSearchInput(BaseModel):
    query: str = Field(..., description="Search query")
    response_format: ResponseFormat = Field(
        default=ResponseFormat.MARKDOWN,
        description="Output format: 'markdown' for human-readable or 'json' for machine-readable"
    )
```

**Markdown 형식**: 헤더, 목록, 사람이 읽을 수 있는 타임스탬프, 괄호 안에 ID가 있는 표시 이름.

**JSON 형식**: 프로그래밍 방식 처리에 적합한 완전하고 구조화된 데이터.

## 페이지네이션

```python
class ListInput(BaseModel):
    limit: Optional[int] = Field(default=20, description="Maximum results to return", ge=1, le=100)
    offset: Optional[int] = Field(default=0, description="Number of results to skip", ge=0)

async def list_items(params: ListInput) -> str:
    data = await api_request(limit=params.limit, offset=params.offset)
    response = {
        "total": data["total"],
        "count": len(data["items"]),
        "offset": params.offset,
        "items": data["items"],
        "has_more": data["total"] > params.offset + len(data["items"]),
        "next_offset": params.offset + len(data["items"]) if data["total"] > params.offset + len(data["items"]) else None
    }
    return json.dumps(response, indent=2)
```

## 에러 처리

```python
def _handle_api_error(e: Exception) -> str:
    if isinstance(e, httpx.HTTPStatusError):
        if e.response.status_code == 404:
            return "Error: Resource not found. Please check the ID is correct."
        elif e.response.status_code == 403:
            return "Error: Permission denied. You don't have access to this resource."
        elif e.response.status_code == 429:
            return "Error: Rate limit exceeded. Please wait before making more requests."
        return f"Error: API request failed with status {e.response.status_code}"
    elif isinstance(e, httpx.TimeoutException):
        return "Error: Request timed out. Please try again."
    return f"Error: Unexpected error occurred: {type(e).__name__}"
```

## 공유 유틸리티

```python
async def _make_api_request(endpoint: str, method: str = "GET", **kwargs) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.request(
            method,
            f"{API_BASE_URL}/{endpoint}",
            timeout=30.0,
            **kwargs
        )
        response.raise_for_status()
        return response.json()
```

## 완전한 예제

```python
#!/usr/bin/env python3
from typing import Optional, List
from enum import Enum
import json
import httpx
from pydantic import BaseModel, Field, field_validator, ConfigDict
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("example_mcp")
API_BASE_URL = "https://api.example.com/v1"

class ResponseFormat(str, Enum):
    MARKDOWN = "markdown"
    JSON = "json"

class UserSearchInput(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True, validate_assignment=True)

    query: str = Field(..., description="Search string to match against names/emails", min_length=2, max_length=200)
    limit: Optional[int] = Field(default=20, description="Maximum results to return", ge=1, le=100)
    offset: Optional[int] = Field(default=0, description="Number of results to skip for pagination", ge=0)
    response_format: ResponseFormat = Field(default=ResponseFormat.MARKDOWN, description="Output format")

    @field_validator('query')
    @classmethod
    def validate_query(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Query cannot be empty or whitespace only")
        return v.strip()

async def _make_api_request(endpoint: str, method: str = "GET", **kwargs) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.request(method, f"{API_BASE_URL}/{endpoint}", timeout=30.0, **kwargs)
        response.raise_for_status()
        return response.json()

def _handle_api_error(e: Exception) -> str:
    if isinstance(e, httpx.HTTPStatusError):
        if e.response.status_code == 404:
            return "Error: Resource not found. Please check the ID is correct."
        elif e.response.status_code == 403:
            return "Error: Permission denied. You don't have access to this resource."
        elif e.response.status_code == 429:
            return "Error: Rate limit exceeded. Please wait before making more requests."
        return f"Error: API request failed with status {e.response.status_code}"
    elif isinstance(e, httpx.TimeoutException):
        return "Error: Request timed out. Please try again."
    return f"Error: Unexpected error occurred: {type(e).__name__}"

@mcp.tool(
    name="example_search_users",
    annotations={
        "title": "Search Example Users",
        "readOnlyHint": True,
        "destructiveHint": False,
        "idempotentHint": True,
        "openWorldHint": True
    }
)
async def example_search_users(params: UserSearchInput) -> str:
    '''Search for users in the Example system by name, email, or team.

    Args:
        params (UserSearchInput): Validated input parameters

    Returns:
        str: JSON or Markdown formatted search results
    '''
    try:
        data = await _make_api_request("users/search", params={
            "q": params.query, "limit": params.limit, "offset": params.offset
        })

        users = data.get("users", [])
        total = data.get("total", 0)

        if not users:
            return f"No users found matching '{params.query}'"

        if params.response_format == ResponseFormat.MARKDOWN:
            lines = [f"# User Search Results: '{params.query}'", "",
                     f"Found {total} users (showing {len(users)})", ""]
            for user in users:
                lines.append(f"## {user['name']} ({user['id']})")
                lines.append(f"- **Email**: {user['email']}")
                if user.get('team'):
                    lines.append(f"- **Team**: {user['team']}")
                lines.append("")
            return "\n".join(lines)
        else:
            response = {
                "total": total,
                "count": len(users),
                "offset": params.offset,
                "users": users
            }
            return json.dumps(response, indent=2)

    except Exception as e:
        return _handle_api_error(e)

if __name__ == "__main__":
    mcp.run()
```

## 고급 FastMCP 기능

### Context 파라미터 주입

```python
from mcp.server.fastmcp import FastMCP, Context

@mcp.tool()
async def advanced_search(query: str, ctx: Context) -> str:
    await ctx.report_progress(0.25, "Starting search...")
    await ctx.log_info("Processing query", {"query": query})
    results = await search_api(query)
    await ctx.report_progress(0.75, "Formatting results...")
    return format_results(results)
```

**Context 기능:**
- `ctx.report_progress(progress, message)` — 긴 작업의 진행상황 보고
- `ctx.log_info/log_error/log_debug(message, data)` — 로깅
- `ctx.elicit(prompt, input_type)` — 사용자 입력 요청
- `ctx.read_resource(uri)` — MCP 리소스 읽기

### 리소스 등록

```python
@mcp.resource("file://documents/{name}")
async def get_document(name: str) -> str:
    '''Expose documents as MCP resources.'''
    with open(f"./docs/{name}", "r") as f:
        return f.read()
```

**리소스 vs 툴**: 리소스는 단순한 URI 기반 파라미터의 데이터 접근에, 툴은 검증과 비즈니스 로직이 있는 복잡한 작업에 사용.

### 수명 관리

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def app_lifespan():
    db = await connect_to_database()
    yield {"db": db}
    await db.close()

mcp = FastMCP("example_mcp", lifespan=app_lifespan)

@mcp.tool()
async def query_data(query: str, ctx: Context) -> str:
    db = ctx.request_context.lifespan_state["db"]
    results = await db.query(query)
    return format_results(results)
```

### 전송 옵션

```python
# stdio (로컬 툴) — 기본값
if __name__ == "__main__":
    mcp.run()

# Streamable HTTP (원격 서버)
if __name__ == "__main__":
    mcp.run(transport="streamable_http", port=8000)
```

## 품질 체크리스트

### 전략적 설계
- [ ] 툴이 단순 API 래퍼가 아닌 완전한 워크플로우를 지원
- [ ] 에러 메시지가 에이전트를 올바른 사용으로 안내

### 구현 품질
- [ ] 모든 툴이 `name`과 `annotations`를 데코레이터에 포함
- [ ] 어노테이션이 올바르게 설정됨
- [ ] 모든 툴이 `Field()` 정의가 있는 Pydantic BaseModel 사용
- [ ] 모든 Pydantic 필드에 명시적 타입과 제약 조건이 있는 설명
- [ ] 모든 툴에 입출력 타입이 명시된 포괄적인 docstring
- [ ] 서버 이름이 `{service}_mcp` 형식을 따름
- [ ] 모든 네트워크 작업이 async/await 사용

### 고급 기능 (해당하는 경우)
- [ ] 긴 작업에 로깅/진행 보고를 위한 Context 주입 사용
- [ ] 적절한 데이터 엔드포인트에 리소스 등록
- [ ] 지속 연결에 수명 관리 구현

### 코드 품질
- [ ] Pydantic 임포트 포함한 적절한 임포트
- [ ] 대용량 결과 세트에 페이지네이션 구현
- [ ] 모든 async 함수가 `async def`로 올바르게 정의
- [ ] HTTP 클라이언트가 적절한 컨텍스트 매니저와 함께 async 패턴 따름
- [ ] 코드 전반에 타입 힌트 사용
- [ ] 모듈 수준에 UPPER_CASE로 상수 정의

### 테스트
- [ ] 서버 실행 성공: `python your_server.py --help`
- [ ] 모든 임포트 정상 해석
- [ ] 예제 툴 호출이 예상대로 동작
- [ ] 에러 시나리오가 적절히 처리됨
