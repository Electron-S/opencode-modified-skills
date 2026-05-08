# 회의록 작성 워크플로우

## 워크플로우

1. STT 텍스트 파일 경로를 받아 Read 툴로 내용을 읽는다
2. `output/minutes/` 하위 **파일명 날짜 기준 최근 3회분** 회의록 파일을 Read 툴로 읽어 용어/맥락을 파악한다
3. 날짜, 제목, 참석자를 순서대로 질문한다
4. **회의 종류 판단**:
   - 제목이 "파트 주간회의"이면 → `output/minutes/파트주간회의/` 저장, 팀원별 업무 보고 구조로 정리
   - 그 외이면 → 회의 배경/사유를 추가로 질문하여 카테고리 파악 후 적절한 하위 폴더에 저장, 안건/주제별 구조로 정리
5. 아래 지침과 출력 형식에 따라 회의록을 작성한다
6. 해당 폴더에 파일로 저장한다 (폴더 없으면 자동 생성)
7. 저장 완료 후 내용을 화면에 출력한다

## 작성 지침

- Speech-to-text 변환본이라 오타/잘못된 인식이 있으니 배경 지식 및 문맥을 고려하여 교정
- **STT 오인식 교정 우선순위**:
  1. `output/minutes/` 하위 최근 3회분 회의록 참고 (파일명 날짜 기준 최신 3개) — 이미 올바르게 교정된 용어, 장비명, 사내 약어, 과제명이 담겨 있음
  2. PMIC 개발 도메인 지식으로 추론 — 아래 배경 지식 활용
- **전문 용어는 영어로 표기** — 한글 발음 표기 금지
  - 장비·측정 용어: Load Transient, Ripple, Efficiency, Oscilloscope, DMM, Source Meter, Spectrum Analyzer 등
  - 반도체 개발 용어: EVT / DVT / MP, Trim, Correlation, Schematic, Layout, Simulation 등
  - 소프트웨어·자동화: GUI, API, DB, Script, Framework, Git, CI/CD 등
  - 제품·과제명: PMIC, SOC, OLED, Vanguard, ATE, EPC 등
- **PMIC 개발 배경 지식 기반 STT 추론**:
  - 평가 항목: Load Transient, Line Transient, Ripple, Efficiency, PSRR, Startup/Shutdown
  - 계측 장비: Oscilloscope, DMM(Digital Multimeter), Source Meter, Electronic Load, Spectrum Analyzer
  - 개발 단계: EVT → DVT → MP
  - 갤럭시 STT 오인식 패턴: 영어 전문 용어를 유사 발음 한글로 변환 (예: "로드 트레이닝" → Load Transient, "뱅가드" → Vanguard)
- 팀원 이름 오인식 시 SKILL.md의 파트원 목록 기준으로 교정
- 불필요한 잡담, 반복 내용 제거
- 불명확한 담당자/마감일은 "확인필요" 표시
- 날짜 언급 없으면 오늘 날짜 사용

## 출력 형식

파일명: `YYYY-MM-DD_제목.md`

```markdown
---
회의 날짜: YYYY-MM-DD
회의 제목: 
참석자: [이름1, 이름2]
---

## Notes
- 핵심 내용만 간결하게 bullet로 정리 (발화자 구분 없이 내용만)
- 중분류로 묶이는 내용은 상위 bullet으로 묶어 처리
- bullet은 최대 3단계

## Action Items
- 할 일 (담당자명 / 마감일)
- 담당자 또는 마감일이 부정확할 경우 "확인필요" 표시
```
