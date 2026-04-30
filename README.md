# opencode-modified-skills

[Anthropic Skills](https://github.com/anthropics/skills)를 [OpenCode](https://github.com/sst/opencode)에서 동작하도록 수정한 모음입니다.

## 배경

Anthropic 공식 스킬들은 일부 기능이 Claude Code CLI(`claude -p`)에 의존하고 있어 OpenCode에서 그대로 사용할 수 없습니다. 이 레포는 해당 의존성을 `opencode run`으로 교체하고, OpenCode의 디렉토리 구조 및 툴명에 맞게 수정한 버전을 관리합니다.

## 포함된 스킬

| 스킬 | 원본 | 주요 변경 사항 |
|---|---|---|
| [skill-creator](./skill-creator/) | [원본](https://github.com/anthropics/skills/tree/main/skills/skill-creator) | `claude -p` → `opencode run --format json`, `.claude/` → `.opencode/` 경로 수정, SKILL.md OpenCode 섹션 추가 |
| [doc-coauthoring](./doc-coauthoring/) | [원본](https://github.com/anthropics/skills/tree/main/skills/doc-coauthoring) | `create_file` → `Write`, `str_replace` → `Edit`, Claude.ai 참조 → OpenCode/MCP 기준으로 수정 |
| [agents-md-management](./agents-md-management/) | [원본](https://github.com/anthropics/claude-plugins-official) (claude-md-management) | 하나의 스킬로 통합, CLAUDE.md → AGENTS.md 타깃으로 변경, opencode 파일 구조 기준으로 수정 |

## 설치 방법

```bash
git clone https://github.com/cyyoo/opencode-modified-skills.git
```

원하는 스킬 폴더를 `~/.config/opencode/skills/` 아래에 복사합니다.

```bash
# 예시: skill-creator 설치
cp -r opencode-modified-skills/skill-creator ~/.config/opencode/skills/skill-creator

# 예시: doc-coauthoring 설치
cp -r opencode-modified-skills/doc-coauthoring ~/.config/opencode/skills/doc-coauthoring

# 예시: agents-md-management 설치
cp -r opencode-modified-skills/agents-md-management ~/.config/opencode/skills/agents-md-management
```

## 사용 방법

설치 후 OpenCode에서 자연어로 요청하면 AI가 자동으로 스킬을 불러옵니다.

```
스킬 새로 하나 만들고 싶어.
```

```
이 기능 명세서 같이 작성해줘.
```

```
오늘 세션에서 배운 거 AGENTS.md에 저장해줘.
```

```
프로젝트 AGENTS.md 상태 점검해줘.
```

## 새 스킬 추가 가이드

다른 Anthropic 스킬을 이 레포에 추가하려면:

1. 원본 스킬 폴더를 레포 루트에 복사
2. `scripts/run_eval.py` — `claude -p` → `opencode run --format json`, `.claude/` → `.opencode/` 경로 수정, JSON 파싱 로직 교체
3. `scripts/improve_description.py` — `claude -p` → `opencode run --format json`
4. `SKILL.md` — `--model` 포맷을 `provider/model-name`으로, Claude.ai/Cowork 섹션을 OpenCode 기준으로 교체
5. 이 README 테이블에 항목 추가

스크립트가 없는 스킬(SKILL.md만 있는 경우)은 툴명(`create_file`→`Write`, `str_replace`→`Edit`)과 Claude.ai URL 참조만 수정하면 됩니다.

`claude-plugins-official` 플러그인(`claude-md-management` 등)을 포팅하는 경우, 플러그인 내 커맨드와 스킬을 하나의 SKILL.md로 통합하고 대상 파일명(`.claude/` 경로 등)을 OpenCode 규칙에 맞게 수정합니다.
