---
name: docx
description: 사용자가 Word 문서(.docx 파일)를 생성, 읽기, 편집, 조작하려 할 때 이 스킬을 사용한다. '워드 문서', '.docx', 목차·제목·페이지 번호·레터헤드 등 서식이 있는 전문 문서를 요청하는 경우가 트리거이다. .docx 파일에서 내용 추출 또는 재구성, 문서에 이미지 삽입 또는 교체, Word 파일에서 찾아 바꾸기, 추적 변경 또는 주석 작업, 내용을 세련된 Word 문서로 변환하는 경우도 해당된다. 사용자가 보고서, 메모, 편지, 템플릿 또는 유사한 결과물을 Word 또는 .docx 파일로 요청하면 이 스킬을 사용한다. PDF, 스프레드시트, Google Docs, 또는 문서 생성과 무관한 일반 코딩 작업에는 사용하지 않는다.
---

# DOCX 생성, 편집, 분석

## 개요

.docx 파일은 XML 파일들을 담은 ZIP 아카이브이다.

## 빠른 참조

| 작업 | 방법 |
|------|------|
| 내용 읽기/분석 | `pandoc` 또는 원시 XML용 unpack |
| 새 문서 생성 | `docx-js` 사용 - 아래 "새 문서 생성" 참고 |
| 기존 문서 편집 | Unpack → XML 편집 → Repack - 아래 "기존 문서 편집" 참고 |

### .doc를 .docx로 변환

레거시 `.doc` 파일은 편집 전에 변환해야 한다:

```bash
python ~/.config/opencode/skills/docx/scripts/office/soffice.py --headless --convert-to docx document.doc
```

### 내용 읽기

```bash
# 추적 변경 사항 포함 텍스트 추출
pandoc --track-changes=all document.docx -o output.md

# 원시 XML 접근
python ~/.config/opencode/skills/docx/scripts/office/unpack.py document.docx unpacked/
```

### 이미지로 변환

```bash
python ~/.config/opencode/skills/docx/scripts/office/soffice.py --headless --convert-to pdf document.docx
pdftoppm -jpeg -r 150 document.pdf page
```

### 추적 변경 사항 수락

LibreOffice를 사용하여 모든 추적 변경 사항이 수락된 깔끔한 문서를 생성한다:

```bash
python ~/.config/opencode/skills/docx/scripts/accept_changes.py input.docx output.docx
```

---

## 새 문서 생성

JavaScript로 .docx 파일을 생성한 후 검증한다. 설치: `npm install -g docx`

### 설정
```javascript
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell, ImageRun,
        Header, Footer, AlignmentType, PageOrientation, LevelFormat, ExternalHyperlink,
        InternalHyperlink, Bookmark, FootnoteReferenceRun, PositionalTab,
        PositionalTabAlignment, PositionalTabRelativeTo, PositionalTabLeader,
        TabStopType, TabStopPosition, Column, SectionType,
        TableOfContents, HeadingLevel, BorderStyle, WidthType, ShadingType,
        VerticalAlign, PageNumber, PageBreak } = require('docx');

const doc = new Document({ sections: [{ children: [/* content */] }] });
Packer.toBuffer(doc).then(buffer => fs.writeFileSync("doc.docx", buffer));
```

### 검증
파일 생성 후 검증한다. 실패 시 unpack하여 XML을 수정하고 repack한다.
```bash
python ~/.config/opencode/skills/docx/scripts/office/validate.py doc.docx
```

### 페이지 크기

```javascript
// 중요: docx-js 기본값은 A4이며 US Letter가 아님
// 일관된 결과를 위해 항상 페이지 크기를 명시적으로 설정
sections: [{
  properties: {
    page: {
      size: {
        width: 12240,   // 8.5인치 (DXA 단위)
        height: 15840   // 11인치 (DXA 단위)
      },
      margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 } // 1인치 여백
    }
  },
  children: [/* content */]
}]
```

**일반 페이지 크기 (DXA 단위, 1440 DXA = 1인치):**

| 용지 | 너비 | 높이 | 내용 너비 (1인치 여백) |
|------|------|------|----------------------|
| US Letter | 12,240 | 15,840 | 9,360 |
| A4 (기본) | 11,906 | 16,838 | 9,026 |

**가로 방향:** docx-js는 내부적으로 너비/높이를 바꾸므로, 세로 방향 크기를 전달하고 스왑을 맡긴다:
```javascript
size: {
  width: 12240,   // 짧은 면을 너비로 전달
  height: 15840,  // 긴 면을 높이로 전달
  orientation: PageOrientation.LANDSCAPE  // docx-js가 XML에서 바꿈
},
```

### 스타일 (내장 제목 재정의)

기본 폰트로 Arial을 사용한다(범용 지원). 가독성을 위해 제목은 검은색으로 유지한다.

```javascript
const doc = new Document({
  styles: {
    default: { document: { run: { font: "Arial", size: 24 } } }, // 기본 12pt
    paragraphStyles: [
      // 중요: 내장 스타일을 재정의하려면 정확한 ID 사용
      { id: "Heading1", name: "Heading 1", basedOn: "Normal", next: "Normal", quickFormat: true,
        run: { size: 32, bold: true, font: "Arial" },
        paragraph: { spacing: { before: 240, after: 240 }, outlineLevel: 0 } }, // TOC를 위해 outlineLevel 필요
      { id: "Heading2", name: "Heading 2", basedOn: "Normal", next: "Normal", quickFormat: true,
        run: { size: 28, bold: true, font: "Arial" },
        paragraph: { spacing: { before: 180, after: 180 }, outlineLevel: 1 } },
    ]
  },
  sections: [{
    children: [
      new Paragraph({ heading: HeadingLevel.HEADING_1, children: [new TextRun("Title")] }),
    ]
  }]
});
```

### 목록 (유니코드 글머리 기호 절대 사용 금지)

```javascript
// ❌ 잘못된 방법 - 글머리 기호 문자를 수동으로 삽입하지 않는다
new Paragraph({ children: [new TextRun("• Item")] })  // 나쁜 예
new Paragraph({ children: [new TextRun("• Item")] })  // 나쁜 예

// ✅ 올바른 방법 - LevelFormat.BULLET과 함께 numbering config 사용
const doc = new Document({
  numbering: {
    config: [
      { reference: "bullets",
        levels: [{ level: 0, format: LevelFormat.BULLET, text: "•", alignment: AlignmentType.LEFT,
          style: { paragraph: { indent: { left: 720, hanging: 360 } } } }] },
      { reference: "numbers",
        levels: [{ level: 0, format: LevelFormat.DECIMAL, text: "%1.", alignment: AlignmentType.LEFT,
          style: { paragraph: { indent: { left: 720, hanging: 360 } } } }] },
    ]
  },
  sections: [{
    children: [
      new Paragraph({ numbering: { reference: "bullets", level: 0 },
        children: [new TextRun("글머리 기호 항목")] }),
      new Paragraph({ numbering: { reference: "numbers", level: 0 },
        children: [new TextRun("번호 항목")] }),
    ]
  }]
});

// ⚠️ 각 reference는 독립된 번호 매기기를 생성함
// 같은 reference = 연속 (1,2,3 → 4,5,6)
// 다른 reference = 재시작 (1,2,3 → 1,2,3)
```

### 표

**중요: 표에는 이중 너비 설정이 필요하다** — 표의 `columnWidths`와 각 셀의 `width` 모두 설정해야 한다. 둘 다 없으면 일부 플랫폼에서 표가 올바르게 렌더링되지 않는다.

```javascript
// 중요: 일관된 렌더링을 위해 항상 표 너비 설정
// 중요: 검은 배경 방지를 위해 ShadingType.CLEAR 사용 (SOLID 아님)
const border = { style: BorderStyle.SINGLE, size: 1, color: "CCCCCC" };
const borders = { top: border, bottom: border, left: border, right: border };

new Table({
  width: { size: 9360, type: WidthType.DXA }, // 항상 DXA 사용 (백분율은 Google Docs에서 깨짐)
  columnWidths: [4680, 4680], // 표 너비의 합과 일치해야 함 (DXA: 1440 = 1인치)
  rows: [
    new TableRow({
      children: [
        new TableCell({
          borders,
          width: { size: 4680, type: WidthType.DXA }, // 각 셀에도 설정
          shading: { fill: "D5E8F0", type: ShadingType.CLEAR }, // SOLID가 아닌 CLEAR
          margins: { top: 80, bottom: 80, left: 120, right: 120 },
          children: [new Paragraph({ children: [new TextRun("Cell")] })]
        })
      ]
    })
  ]
})
```

**표 너비 계산:**

항상 `WidthType.DXA` 사용 — `WidthType.PERCENTAGE`는 Google Docs에서 깨진다.

```javascript
// US Letter 1인치 여백: 12240 - 2880 = 9360 DXA
width: { size: 9360, type: WidthType.DXA },
columnWidths: [7000, 2360]  // 표 너비와 합산이 일치해야 함
```

### 이미지

```javascript
// 중요: type 파라미터 필수
new Paragraph({
  children: [new ImageRun({
    type: "png", // 필수: png, jpg, jpeg, gif, bmp, svg
    data: fs.readFileSync("image.png"),
    transformation: { width: 200, height: 150 },
    altText: { title: "Title", description: "Desc", name: "Name" } // 세 가지 모두 필수
  })]
})
```

### 페이지 나누기

```javascript
// 중요: PageBreak는 Paragraph 안에 있어야 함
new Paragraph({ children: [new PageBreak()] })

// 또는 pageBreakBefore 사용
new Paragraph({ pageBreakBefore: true, children: [new TextRun("새 페이지")] })
```

### 하이퍼링크

```javascript
// 외부 링크
new Paragraph({
  children: [new ExternalHyperlink({
    children: [new TextRun({ text: "여기를 클릭", style: "Hyperlink" })],
    link: "https://example.com",
  })]
})

// 내부 링크 (북마크 + 참조)
// 1. 목적지에 북마크 생성
new Paragraph({ heading: HeadingLevel.HEADING_1, children: [
  new Bookmark({ id: "chapter1", children: [new TextRun("Chapter 1")] }),
]})
// 2. 링크 연결
new Paragraph({ children: [new InternalHyperlink({
  children: [new TextRun({ text: "Chapter 1 참고", style: "Hyperlink" })],
  anchor: "chapter1",
})]})
```

### 각주

```javascript
const doc = new Document({
  footnotes: {
    1: { children: [new Paragraph("Source: Annual Report 2024")] },
    2: { children: [new Paragraph("See appendix for methodology")] },
  },
  sections: [{
    children: [new Paragraph({
      children: [
        new TextRun("Revenue grew 15%"),
        new FootnoteReferenceRun(1),
        new TextRun(" using adjusted metrics"),
        new FootnoteReferenceRun(2),
      ],
    })]
  }]
});
```

### 탭 정지

```javascript
// 같은 줄에서 텍스트 오른쪽 정렬 (예: 날짜와 제목 반대편)
new Paragraph({
  children: [
    new TextRun("회사명"),
    new TextRun("\t2025년 1월"),
  ],
  tabStops: [{ type: TabStopType.RIGHT, position: TabStopPosition.MAX }],
})
```

### 다단 레이아웃

```javascript
// 동일 너비 단
sections: [{
  properties: {
    column: {
      count: 2,
      space: 720,        // 단 간격 DXA (720 = 0.5인치)
      equalWidth: true,
      separate: true,    // 단 사이 세로선
    },
  },
  children: [/* 내용이 단에 자연스럽게 흐름 */]
}]
```

### 목차

```javascript
// 중요: 제목에는 HeadingLevel만 사용 - 사용자 정의 스타일 사용 금지
new TableOfContents("목차", { hyperlink: true, headingStyleRange: "1-3" })
```

### 머리글/바닥글

```javascript
sections: [{
  properties: {
    page: { margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 } }
  },
  headers: {
    default: new Header({ children: [new Paragraph({ children: [new TextRun("헤더")] })] })
  },
  footers: {
    default: new Footer({ children: [new Paragraph({
      children: [new TextRun("Page "), new TextRun({ children: [PageNumber.CURRENT] })]
    })] })
  },
  children: [/* content */]
}]
```

### docx-js 핵심 규칙

- **페이지 크기를 명시적으로 설정** - docx-js 기본은 A4; 미국 문서에는 US Letter 사용
- **가로 방향: 세로 방향 크기를 전달** - docx-js가 내부적으로 너비/높이 교환
- **`\n` 절대 사용 금지** - 별도 Paragraph 요소 사용
- **유니코드 글머리 기호 사용 금지** - numbering config와 `LevelFormat.BULLET` 사용
- **PageBreak는 Paragraph 안에** - 독립 사용 시 잘못된 XML 생성
- **ImageRun에 `type` 필수** - 항상 png/jpg 등 지정
- **표 `width`에 항상 DXA 사용** - `WidthType.PERCENTAGE` 사용 금지
- **표에 이중 너비 필요** - `columnWidths` 배열과 셀 `width` 모두 설정
- **`ShadingType.CLEAR` 사용** - 표 음영에 SOLID 사용 금지
- **표를 구분선으로 사용 금지** - 대신 Paragraph에 `border: { bottom: ... }` 사용
- **TOC는 HeadingLevel만 사용** - 제목 단락에 사용자 정의 스타일 금지
- **내장 스타일 재정의** - 정확한 ID 사용: "Heading1", "Heading2" 등
- **`outlineLevel` 포함** - TOC를 위해 필수 (H1=0, H2=1 등)

---

## 기존 문서 편집

**3단계를 순서대로 따른다.**

### 1단계: Unpack
```bash
python ~/.config/opencode/skills/docx/scripts/office/unpack.py document.docx unpacked/
```
XML 추출, 예쁘게 인쇄, 인접 런 병합, 스마트 인용 부호를 XML 엔티티로 변환한다.

### 2단계: XML 편집

`unpacked/word/`의 파일을 편집한다.

**추적 변경 사항과 주석에는 "Claude"를 작성자로 사용한다** (사용자가 다른 이름을 명시하지 않는 경우).

**문자열 교체에는 Edit 도구를 직접 사용한다. Python 스크립트를 작성하지 않는다.** 스크립트는 불필요한 복잡성을 추가한다. Edit 도구는 교체 대상을 정확히 보여준다.

**중요: 새 내용에는 스마트 인용 부호를 사용한다.** 따옴표나 아포스트로피가 있는 텍스트 추가 시 XML 엔티티를 사용한다:
```xml
<w:t>Here&#x2019;s a quote: &#x201C;Hello&#x201D;</w:t>
```
| 엔티티 | 문자 |
|--------|------|
| `&#x2018;` | ' (왼쪽 작은따옴표) |
| `&#x2019;` | ' (오른쪽 작은따옴표/아포스트로피) |
| `&#x201C;` | " (왼쪽 큰따옴표) |
| `&#x201D;` | " (오른쪽 큰따옴표) |

**주석 추가:** 여러 XML 파일에 걸친 보일러플레이트 처리에는 `comment.py` 사용:
```bash
python ~/.config/opencode/skills/docx/scripts/comment.py unpacked/ 0 "Comment text"
python ~/.config/opencode/skills/docx/scripts/comment.py unpacked/ 1 "Reply text" --parent 0
python ~/.config/opencode/skills/docx/scripts/comment.py unpacked/ 0 "Text" --author "Custom Author"
```

### 3단계: Pack
```bash
python ~/.config/opencode/skills/docx/scripts/office/pack.py unpacked/ output.docx --original document.docx
```
검증, 자동 복구, XML 압축, DOCX 생성.

**자동 복구 대상:**
- `durableId` >= 0x7FFFFFFF (유효한 ID 재생성)
- 공백이 있는 `<w:t>`의 누락된 `xml:space="preserve"`

**자동 복구 불가:**
- 잘못된 XML, 잘못된 요소 중첩, 누락된 관계, 스키마 위반

---

## XML 참조

### 추적 변경 사항

**삽입:**
```xml
<w:ins w:id="1" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:t>삽입된 텍스트</w:t></w:r>
</w:ins>
```

**삭제:**
```xml
<w:del w:id="2" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:delText>삭제된 텍스트</w:delText></w:r>
</w:del>
```

**최소 편집** - 변경되는 부분만 표시:
```xml
<w:r><w:t>The term is </w:t></w:r>
<w:del w:id="1" w:author="Claude" w:date="...">
  <w:r><w:delText>30</w:delText></w:r>
</w:del>
<w:ins w:id="2" w:author="Claude" w:date="...">
  <w:r><w:t>60</w:t></w:r>
</w:ins>
<w:r><w:t> days.</w:t></w:r>
```

### 주석

`comment.py` 실행 후(2단계 참고) document.xml에 마커를 추가한다.

**중요: `<w:commentRangeStart>`와 `<w:commentRangeEnd>`는 `<w:r>`의 형제 요소이며, `<w:r>` 내부에 있으면 안 된다.**

```xml
<w:commentRangeStart w:id="0"/>
<w:r><w:t>텍스트</w:t></w:r>
<w:commentRangeEnd w:id="0"/>
<w:r><w:rPr><w:rStyle w:val="CommentReference"/></w:rPr><w:commentReference w:id="0"/></w:r>
```

### 이미지

1. `word/media/`에 이미지 파일 추가
2. `word/_rels/document.xml.rels`에 관계 추가:
```xml
<Relationship Id="rId5" Type=".../image" Target="media/image1.png"/>
```
3. `[Content_Types].xml`에 콘텐츠 유형 추가:
```xml
<Default Extension="png" ContentType="image/png"/>
```
4. document.xml에서 참조:
```xml
<w:drawing>
  <wp:inline>
    <wp:extent cx="914400" cy="914400"/>  <!-- EMU: 914400 = 1인치 -->
    <a:graphic>
      <a:graphicData uri=".../picture">
        <pic:pic>
          <pic:blipFill><a:blip r:embed="rId5"/></pic:blipFill>
        </pic:pic>
      </a:graphicData>
    </a:graphic>
  </wp:inline>
</w:drawing>
```

---

## 의존성

- **pandoc**: 텍스트 추출
- **docx**: `npm install -g docx` (새 문서 생성)
- **LibreOffice**: PDF 변환 (`scripts/office/soffice.py`로 자동 설정)
- **Poppler**: 이미지용 `pdftoppm`
