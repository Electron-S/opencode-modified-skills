# opencode-modified-skills

Claude Code용으로 만들어진 [Anthropic Skills](https://github.com/anthropics/skills)를 OpenCode에서 동작하도록 수정한 모음입니다.

## 배경

Anthropic의 공식 스킬들은 Claude Code CLI(`claude -p`)에 의존하는 부분이 있어 OpenCode에서 그대로 사용할 수 없습니다. 이 레포는 해당 의존성을 `opencode run`으로 교체하고 OpenCode의 디렉토리 구조에 맞게 수정한 버전을 관리합니다.

## 포함된 스킬

| 스킬 | 원본 | 주요 변경 사항 |
|---|---|---|
| [skill-creator](./skill-creator/) | [anthropics/skills](https://github.com/anthropics/skills/tree/main/skills/skill-creator) | `claude -p` → `opencode run`, `.claude/` → `.opencode/` 경로 수정, SKILL.md OpenCode 섹션 추가 |

## 설치 방법

각 스킬 폴더를 `~/.config/opencode/skills/` 아래에 복사하면 됩니다.

**전체 설치:**
```bash
git clone https://github.com/<your-username>/opencode-modified-skills.git
cp -r opencode-modified-skills/skill-creator ~/.config/opencode/skills/
```

**특정 스킬만 설치:**
```bash
cp -r opencode-modified-skills/skill-creator ~/.config/opencode/skills/skill-creator
```

## 사용 방법

설치 후 OpenCode에서 자연어로 요청하면 AI가 자동으로 스킬을 불러옵니다.

```
스킬 새로 하나 만들고 싶어.
```

## 스킬 추가 방법

1. `skills/` 아래에 원본 스킬 폴더를 복사
2. `scripts/run_eval.py` — `claude -p` → `opencode run --format json` 교체, `.claude/` → `.opencode/` 경로 수정
3. `scripts/improve_description.py` — `claude -p` → `opencode run --format json` 교체
4. `SKILL.md` — `--model` 포맷을 `provider/model-name`으로 수정, Claude.ai/Cowork 섹션을 OpenCode 섹션으로 교체
5. README 테이블에 추가
