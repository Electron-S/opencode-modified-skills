# Node/TypeScript MCP 서버 구현 가이드

## 개요

TypeScript SDK를 사용해 MCP 서버를 구축하는 Node/TypeScript 전용 모범 사례 및 예제. 프로젝트 구조, 서버 설정, 툴 등록 패턴, Zod 입력 검증, 에러 처리, 완전한 예제 포함.

---

## 빠른 참조

### 핵심 임포트
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import express from "express";
import { z } from "zod";
```

### 서버 초기화
```typescript
const server = new McpServer({
  name: "service-mcp-server",
  version: "1.0.0"
});
```

### 툴 등록 패턴
```typescript
server.registerTool(
  "tool_name",
  {
    title: "Tool Display Name",
    description: "What the tool does",
    inputSchema: { param: z.string() },
    outputSchema: { result: z.string() }
  },
  async ({ param }) => {
    const output = { result: `Processed: ${param}` };
    return {
      content: [{ type: "text", text: JSON.stringify(output) }],
      structuredContent: output
    };
  }
);
```

---

## MCP TypeScript SDK

공식 MCP TypeScript SDK가 제공하는 것:
- 서버 초기화용 `McpServer` 클래스
- 툴 등록용 `registerTool` 메서드
- 런타임 입력 검증을 위한 Zod 스키마 통합
- 타입 안전한 툴 핸들러 구현

**중요 - 최신 API만 사용:**
- **사용할 것**: `server.registerTool()`, `server.registerResource()`, `server.registerPrompt()`
- **사용하지 말 것**: `server.tool()`, `server.setRequestHandler(ListToolsRequestSchema, ...)`, 수동 핸들러 등록
- `register*` 메서드가 더 나은 타입 안전성, 자동 스키마 처리를 제공하며 권장 방법

## 서버 이름 규칙

Node/TypeScript MCP 서버 이름 패턴:
- **형식**: `{service}-mcp-server` (소문자, 하이픈)
- **예**: `github-mcp-server`, `jira-mcp-server`, `stripe-mcp-server`

## 프로젝트 구조

```
{service}-mcp-server/
├── package.json
├── tsconfig.json
├── README.md
├── src/
│   ├── index.ts          # McpServer 초기화가 있는 메인 진입점
│   ├── types.ts          # TypeScript 타입 정의 및 인터페이스
│   ├── tools/            # 툴 구현 (도메인별 파일)
│   ├── services/         # API 클라이언트 및 공유 유틸리티
│   ├── schemas/          # Zod 검증 스키마
│   └── constants.ts      # 공유 상수 (API_URL, CHARACTER_LIMIT 등)
└── dist/                 # 빌드된 JavaScript 파일 (진입점: dist/index.js)
```

## 툴 구현

### 툴 이름

snake_case 사용 (예: `search_users`, `create_project`, `get_channel_info`).

**이름 충돌 방지**: 서비스 컨텍스트 포함:
- `send_message` 대신 `slack_send_message`
- `create_issue` 대신 `github_create_issue`
- `list_tasks` 대신 `asana_list_tasks`

### 툴 구조

`registerTool` 메서드로 툴 등록. 요구사항:
- 런타임 입력 검증 및 타입 안전성을 위해 Zod 스키마 사용
- `description` 필드를 명시적으로 제공 (JSDoc 주석은 자동 추출되지 않음)
- `title`, `description`, `inputSchema`, `annotations` 명시적 제공
- `inputSchema`는 JSON 스키마가 아닌 Zod 스키마 객체

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({
  name: "example-mcp",
  version: "1.0.0"
});

const UserSearchInputSchema = z.object({
  query: z.string()
    .min(2, "Query must be at least 2 characters")
    .max(200, "Query must not exceed 200 characters")
    .describe("Search string to match against names/emails"),
  limit: z.number()
    .int()
    .min(1)
    .max(100)
    .default(20)
    .describe("Maximum results to return"),
  offset: z.number()
    .int()
    .min(0)
    .default(0)
    .describe("Number of results to skip for pagination"),
  response_format: z.nativeEnum(ResponseFormat)
    .default(ResponseFormat.MARKDOWN)
    .describe("Output format: 'markdown' for human-readable or 'json' for machine-readable")
}).strict();

type UserSearchInput = z.infer<typeof UserSearchInputSchema>;

server.registerTool(
  "example_search_users",
  {
    title: "Search Example Users",
    description: `Search for users in the Example system by name, email, or team.

Args:
  - query (string): Search string to match against names/emails
  - limit (number): Maximum results to return, between 1-100 (default: 20)
  - offset (number): Number of results to skip for pagination (default: 0)
  - response_format ('markdown' | 'json'): Output format (default: 'markdown')

Returns JSON schema:
  {
    "total": number,
    "count": number,
    "offset": number,
    "users": [{ "id": string, "name": string, "email": string, "team"?: string }],
    "has_more": boolean,
    "next_offset"?: number
  }`,
    inputSchema: UserSearchInputSchema,
    annotations: {
      readOnlyHint: true,
      destructiveHint: false,
      idempotentHint: true,
      openWorldHint: true
    }
  },
  async (params: UserSearchInput) => {
    try {
      const data = await makeApiRequest<any>("users/search", "GET", undefined, {
        q: params.query,
        limit: params.limit,
        offset: params.offset
      });

      const users = data.users || [];
      const total = data.total || 0;

      if (!users.length) {
        return { content: [{ type: "text", text: `No users found matching '${params.query}'` }] };
      }

      const output = {
        total,
        count: users.length,
        offset: params.offset,
        users: users.map((user: any) => ({
          id: user.id,
          name: user.name,
          email: user.email,
          ...(user.team ? { team: user.team } : {}),
          active: user.active ?? true
        })),
        has_more: total > params.offset + users.length,
        ...(total > params.offset + users.length ? { next_offset: params.offset + users.length } : {})
      };

      let textContent: string;
      if (params.response_format === ResponseFormat.MARKDOWN) {
        const lines = [`# User Search Results: '${params.query}'`, "", `Found ${total} users (showing ${users.length})`, ""];
        for (const user of users) {
          lines.push(`## ${user.name} (${user.id})`);
          lines.push(`- **Email**: ${user.email}`);
          if (user.team) lines.push(`- **Team**: ${user.team}`);
          lines.push("");
        }
        textContent = lines.join("\n");
      } else {
        textContent = JSON.stringify(output, null, 2);
      }

      return {
        content: [{ type: "text", text: textContent }],
        structuredContent: output
      };
    } catch (error) {
      return { content: [{ type: "text", text: handleApiError(error) }] };
    }
  }
);
```

## Zod 스키마

```typescript
import { z } from "zod";

const CreateUserSchema = z.object({
  name: z.string().min(1, "Name is required").max(100),
  email: z.string().email("Invalid email format"),
  age: z.number().int().min(0).max(150)
}).strict();  // .strict()으로 추가 필드 금지

enum ResponseFormat {
  MARKDOWN = "markdown",
  JSON = "json"
}

const SearchSchema = z.object({
  response_format: z.nativeEnum(ResponseFormat).default(ResponseFormat.MARKDOWN)
});

const PaginationSchema = z.object({
  limit: z.number().int().min(1).max(100).default(20),
  offset: z.number().int().min(0).default(0)
});
```

## 페이지네이션

```typescript
const ListSchema = z.object({
  limit: z.number().int().min(1).max(100).default(20),
  offset: z.number().int().min(0).default(0)
});

async function listItems(params: z.infer<typeof ListSchema>) {
  const data = await apiRequest(params.limit, params.offset);
  return JSON.stringify({
    total: data.total,
    count: data.items.length,
    offset: params.offset,
    items: data.items,
    has_more: data.total > params.offset + data.items.length,
    next_offset: data.total > params.offset + data.items.length
      ? params.offset + data.items.length : undefined
  }, null, 2);
}
```

## 응답 크기 제한

```typescript
export const CHARACTER_LIMIT = 25000;

async function searchTool(params: SearchInput) {
  let result = generateResponse(data);
  if (result.length > CHARACTER_LIMIT) {
    const truncatedData = data.slice(0, Math.max(1, data.length / 2));
    response.data = truncatedData;
    response.truncated = true;
    response.truncation_message =
      `Response truncated from ${data.length} to ${truncatedData.length} items. ` +
      `Use 'offset' parameter or add filters to see more results.`;
    result = JSON.stringify(response, null, 2);
  }
  return result;
}
```

## 에러 처리

```typescript
import axios, { AxiosError } from "axios";

function handleApiError(error: unknown): string {
  if (error instanceof AxiosError) {
    if (error.response) {
      switch (error.response.status) {
        case 404: return "Error: Resource not found. Please check the ID is correct.";
        case 403: return "Error: Permission denied. You don't have access to this resource.";
        case 429: return "Error: Rate limit exceeded. Please wait before making more requests.";
        default: return `Error: API request failed with status ${error.response.status}`;
      }
    } else if (error.code === "ECONNABORTED") {
      return "Error: Request timed out. Please try again.";
    }
  }
  return `Error: Unexpected error occurred: ${error instanceof Error ? error.message : String(error)}`;
}
```

## 공유 유틸리티

```typescript
async function makeApiRequest<T>(
  endpoint: string,
  method: "GET" | "POST" | "PUT" | "DELETE" = "GET",
  data?: any,
  params?: any
): Promise<T> {
  const response = await axios({
    method,
    url: `${API_BASE_URL}/${endpoint}`,
    data,
    params,
    timeout: 30000,
    headers: { "Content-Type": "application/json", "Accept": "application/json" }
  });
  return response.data;
}
```

## 패키지 설정

### package.json
```json
{
  "name": "{service}-mcp-server",
  "version": "1.0.0",
  "description": "MCP server for {Service} API integration",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "clean": "rm -rf dist"
  },
  "engines": { "node": ">=18" },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.6.1",
    "axios": "^1.7.9",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/node": "^22.10.0",
    "tsx": "^4.19.2",
    "typescript": "^5.7.2"
  }
}
```

### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "allowSyntheticDefaultImports": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## 전송 설정

### stdio (로컬 서버)
```typescript
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

async function runStdio() {
  if (!process.env.EXAMPLE_API_KEY) {
    console.error("ERROR: EXAMPLE_API_KEY environment variable is required");
    process.exit(1);
  }
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("MCP server running via stdio");
}
```

### Streamable HTTP (원격 서버)
```typescript
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";

async function runHTTP() {
  const app = express();
  app.use(express.json());

  app.post('/mcp', async (req, res) => {
    const transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: undefined,
      enableJsonResponse: true
    });
    res.on('close', () => transport.close());
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
  });

  const port = parseInt(process.env.PORT || '3000');
  app.listen(port, () => console.error(`MCP server running on http://localhost:${port}/mcp`));
}

const transport = process.env.TRANSPORT || 'stdio';
if (transport === 'http') {
  runHTTP().catch(error => { console.error("Server error:", error); process.exit(1); });
} else {
  runStdio().catch(error => { console.error("Server error:", error); process.exit(1); });
}
```

## 고급 기능

### 리소스 등록

```typescript
server.registerResource(
  {
    uri: "file://documents/{name}",
    name: "Document Resource",
    description: "Access documents by name",
    mimeType: "text/plain"
  },
  async (uri: string) => {
    const match = uri.match(/^file:\/\/documents\/(.+)$/);
    if (!match) throw new Error("Invalid URI format");
    const content = await loadDocument(match[1]);
    return { contents: [{ uri, mimeType: "text/plain", text: content }] };
  }
);
```

**리소스 vs 툴 선택 기준:**
- **리소스**: URI 기반 파라미터로 데이터 접근, 정적/반정적 데이터
- **툴**: 검증과 비즈니스 로직이 필요한 복잡한 작업, 사이드 이펙트 있는 작업

## 빌드 및 실행

```bash
npm run build    # dist/ 폴더로 컴파일
npm start        # dist/index.js 실행
npm run dev      # 자동 리로드로 개발 실행
```

`npm run build`가 성공적으로 완료되어야 구현 완료로 간주한다.

## 품질 체크리스트

### 전략적 설계
- [ ] 툴이 단순 API 래퍼가 아닌 완전한 워크플로우를 지원
- [ ] 툴 이름이 자연스러운 작업 구분을 반영
- [ ] 응답 형식이 에이전트 컨텍스트 효율성에 최적화
- [ ] 에러 메시지가 에이전트를 올바른 사용으로 안내

### 구현 품질
- [ ] 모든 툴이 `registerTool`로 완전한 설정과 함께 등록됨
- [ ] `title`, `description`, `inputSchema`, `annotations` 모두 포함
- [ ] 어노테이션이 올바르게 설정됨
- [ ] 모든 툴이 `.strict()` 적용된 Zod 스키마 사용
- [ ] 모든 설명이 입출력 타입과 예제를 포함

### TypeScript 품질
- [ ] 모든 데이터 구조에 TypeScript 인터페이스 정의
- [ ] tsconfig.json에 strict 모드 활성화
- [ ] `any` 타입 사용 없음 — `unknown` 또는 적절한 타입 사용
- [ ] 모든 async 함수에 명시적 `Promise<T>` 반환 타입
- [ ] 에러 처리에 적절한 타입 가드 사용

### 프로젝트 설정
- [ ] package.json에 모든 필요한 의존성 포함
- [ ] 빌드 스크립트가 dist/에 JavaScript 생성
- [ ] 서버 이름이 `{service}-mcp-server` 형식을 따름
- [ ] `npm run build`가 오류 없이 성공적으로 완료

### 코드 품질
- [ ] 페이지네이션이 적절히 구현됨
- [ ] 큰 응답이 CHARACTER_LIMIT을 확인하고 명확한 메시지와 함께 잘림
- [ ] 공통 기능이 재사용 가능한 함수로 추출됨
- [ ] 모든 네트워크 작업이 타임아웃과 연결 에러를 처리
