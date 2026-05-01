# 프레젠테이션 편집

## 템플릿 기반 워크플로

기존 프레젠테이션을 템플릿으로 사용할 때:

1. **기존 슬라이드 분석**:
   ```bash
   python ~/.config/opencode/skills/pptx/scripts/thumbnail.py template.pptx
   python -m markitdown template.pptx
   ```
   `thumbnails.jpg`에서 레이아웃 확인, markitdown 출력에서 플레이스홀더 텍스트 확인.

2. **슬라이드 매핑 계획**: 각 콘텐츠 섹션에 맞는 템플릿 슬라이드를 선택한다.

   ⚠️ **다양한 레이아웃 사용** — 단조로운 프레젠테이션은 흔한 실패 유형이다. 기본 제목 + 글머리 기호 슬라이드로 기본 설정하지 않는다. 적극적으로 다음을 찾는다:
   - 다단 레이아웃 (2단, 3단)
   - 이미지 + 텍스트 조합
   - 텍스트 오버레이가 있는 전체 블리드 이미지
   - 인용 또는 콜아웃 슬라이드
   - 섹션 구분자
   - 통계/숫자 콜아웃
   - 아이콘 그리드 또는 아이콘 + 텍스트 행

   **피해야 할 것:** 모든 슬라이드에 같은 텍스트 중심 레이아웃 반복.

   콘텐츠 유형에 레이아웃 스타일을 맞춘다 (예: 주요 사항 → 글머리 기호, 팀 정보 → 다단, 추천사 → 인용 슬라이드).

3. **Unpack**: `python ~/.config/opencode/skills/pptx/scripts/office/unpack.py template.pptx unpacked/`

4. **프레젠테이션 구성** (서브에이전트가 아닌 직접 수행):
   - 원하지 않는 슬라이드 삭제 (`<p:sldIdLst>`에서 제거)
   - 재사용할 슬라이드 복제 (`add_slide.py`)
   - `<p:sldIdLst>`에서 슬라이드 순서 변경
   - **5단계 전에 모든 구조적 변경 완료**

5. **내용 편집**: 각 `slide{N}.xml`의 텍스트 업데이트.
   **서브에이전트가 있으면 여기서 사용** — 슬라이드는 별개의 XML 파일이므로 병렬 편집 가능.

6. **Clean**: `python ~/.config/opencode/skills/pptx/scripts/clean.py unpacked/`

7. **Pack**: `python ~/.config/opencode/skills/pptx/scripts/office/pack.py unpacked/ output.pptx --original template.pptx`

---

## 스크립트

| 스크립트 | 목적 |
|---------|------|
| `unpack.py` | PPTX 추출 및 예쁘게 인쇄 |
| `add_slide.py` | 슬라이드 복제 또는 레이아웃에서 생성 |
| `clean.py` | 고아 파일 제거 |
| `pack.py` | 검증과 함께 재패킹 |
| `thumbnail.py` | 슬라이드 시각적 그리드 생성 |

### unpack.py

```bash
python ~/.config/opencode/skills/pptx/scripts/office/unpack.py input.pptx unpacked/
```

PPTX 추출, XML 예쁘게 인쇄, 스마트 인용 부호 이스케이프.

### add_slide.py

```bash
python ~/.config/opencode/skills/pptx/scripts/add_slide.py unpacked/ slide2.xml      # 슬라이드 복제
python ~/.config/opencode/skills/pptx/scripts/add_slide.py unpacked/ slideLayout2.xml # 레이아웃에서 생성
```

원하는 위치의 `<p:sldIdLst>`에 추가할 `<p:sldId>`를 출력한다.

### clean.py

```bash
python ~/.config/opencode/skills/pptx/scripts/clean.py unpacked/
```

`<p:sldIdLst>`에 없는 슬라이드, 참조되지 않는 미디어, 고아 rels를 제거한다.

### pack.py

```bash
python ~/.config/opencode/skills/pptx/scripts/office/pack.py unpacked/ output.pptx --original input.pptx
```

검증, 복구, XML 압축, 스마트 인용 부호 재인코딩.

### thumbnail.py

```bash
python ~/.config/opencode/skills/pptx/scripts/thumbnail.py input.pptx [output_prefix] [--cols N]
```

슬라이드 파일명을 레이블로 하는 `thumbnails.jpg` 생성. 기본 3열, 그리드당 최대 12개.

**템플릿 분석용으로만 사용** (레이아웃 선택). 시각적 QA에는 `soffice` + `pdftoppm`으로 전체 해상도 개별 슬라이드 이미지 생성을 사용한다(SKILL.md 참고).

---

## 슬라이드 작업

슬라이드 순서는 `ppt/presentation.xml` → `<p:sldIdLst>`에 있다.

**순서 변경**: `<p:sldId>` 요소 재배치.

**삭제**: `<p:sldId>` 제거 후 `clean.py` 실행.

**추가**: `add_slide.py` 사용. 슬라이드 파일을 수동으로 복사하지 않는다 — 스크립트가 노트 참조, Content_Types.xml, 관계 ID를 처리한다.

---

## 내용 편집

**서브에이전트:** 가능하면 여기서 사용한다(4단계 완료 후). 각 슬라이드는 별개의 XML 파일이므로 병렬 편집 가능. 서브에이전트 프롬프트에 포함할 사항:
- 편집할 슬라이드 파일 경로
- **"모든 변경에 Edit 도구를 사용한다"**
- 아래 서식 규칙과 일반적인 함정

각 슬라이드에서:
1. 슬라이드 XML 읽기
2. 모든 플레이스홀더 내용 파악 — 텍스트, 이미지, 차트, 아이콘, 캡션
3. 각 플레이스홀더를 최종 내용으로 교체

**sed나 Python 스크립트가 아닌 Edit 도구를 사용한다.** Edit 도구는 교체 대상을 정확히 지정하도록 강제하여 신뢰성을 높인다.

### 서식 규칙

- **모든 헤더, 소제목, 인라인 레이블을 굵게**: `<a:rPr>`에 `b="1"` 사용. 포함 사항:
  - 슬라이드 제목
  - 슬라이드 내 섹션 헤더
  - 줄 시작 부분의 인라인 레이블 (예: "Status:", "설명:")
- **유니코드 글머리 기호(•) 절대 사용 금지**: `<a:buChar>` 또는 `<a:buAutoNum>`으로 올바른 목록 서식 사용
- **글머리 기호 일관성**: 레이아웃에서 상속받도록 한다. `<a:buChar>` 또는 `<a:buNone>`만 지정.

---

## 일반적인 함정

### 템플릿 적용

소스 내용이 템플릿보다 항목이 적은 경우:
- **초과 요소를 완전히 제거** (이미지, 도형, 텍스트 박스), 텍스트만 지우지 않는다
- 텍스트 내용 지운 후 고아 시각 요소 확인
- 불일치하는 개수를 포착하기 위해 시각적 QA 실행

내용을 다른 길이의 텍스트로 교체할 때:
- **더 짧은 교체**: 보통 안전
- **더 긴 교체**: 오버플로 또는 예상치 못한 줄바꿈 가능
- 텍스트 변경 후 시각적 QA로 테스트
- 템플릿의 디자인 제약에 맞게 내용 잘라내거나 분리 고려

**템플릿 슬롯 ≠ 소스 항목**: 템플릿에 팀원 4명이 있지만 소스에는 3명이면, 4번째 팀원의 그룹 전체(이미지 + 텍스트 박스) 삭제.

### 다중 항목 내용

소스에 여러 항목(번호 목록, 여러 섹션)이 있으면 각각 별개의 `<a:p>` 요소 생성 — **하나의 문자열로 연결하지 않는다**.

**❌ 잘못된 방법** — 모든 항목을 하나의 단락에:
```xml
<a:p>
  <a:r><a:rPr .../><a:t>1단계: 첫 번째 작업. 2단계: 두 번째 작업.</a:t></a:r>
</a:p>
```

**✅ 올바른 방법** — 굵은 헤더가 있는 별개 단락:
```xml
<a:p>
  <a:pPr algn="l"><a:lnSpc><a:spcPts val="3919"/></a:lnSpc></a:pPr>
  <a:r><a:rPr lang="ko-KR" sz="2799" b="1" .../><a:t>1단계</a:t></a:r>
</a:p>
<a:p>
  <a:pPr algn="l"><a:lnSpc><a:spcPts val="3919"/></a:lnSpc></a:pPr>
  <a:r><a:rPr lang="ko-KR" sz="2799" .../><a:t>첫 번째 작업을 수행한다.</a:t></a:r>
</a:p>
```

행 간격 보존을 위해 원본 단락의 `<a:pPr>`를 복사한다. 헤더에는 `b="1"` 사용.

### 스마트 인용 부호

unpack/pack이 자동으로 처리하지만 Edit 도구는 스마트 인용 부호를 ASCII로 변환한다.

**인용 부호가 있는 새 텍스트 추가 시 XML 엔티티 사용:**

```xml
<a:t>the &#x201C;Agreement&#x201D;</a:t>
```

| 문자 | 이름 | Unicode | XML 엔티티 |
|------|------|---------|-----------|
| `"` | 왼쪽 큰따옴표 | U+201C | `&#x201C;` |
| `"` | 오른쪽 큰따옴표 | U+201D | `&#x201D;` |
| `'` | 왼쪽 작은따옴표 | U+2018 | `&#x2018;` |
| `'` | 오른쪽 작은따옴표 | U+2019 | `&#x2019;` |

### 기타

- **공백**: 선행/후행 공백이 있는 `<a:t>`에 `xml:space="preserve"` 사용
- **XML 파싱**: `xml.etree.ElementTree`(네임스페이스 손상) 대신 `defusedxml.minidom` 사용
