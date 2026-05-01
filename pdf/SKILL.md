---
name: pdf
description: PDF 파일과 관련된 모든 작업에 이 스킬을 사용한다. PDF에서 텍스트·표 읽기 및 추출, 여러 PDF 합치기, PDF 분할, 페이지 회전, 워터마크 추가, 새 PDF 생성, PDF 양식 채우기, 암호화/복호화, 이미지 추출, 스캔 PDF OCR 등이 모두 해당된다. 사용자가 .pdf 파일을 언급하거나 PDF를 생성해 달라고 요청하면 이 스킬을 사용한다.
---

# PDF 처리 가이드

## 개요

이 가이드는 Python 라이브러리와 커맨드라인 도구를 이용한 핵심 PDF 처리 작업을 다룬다. 고급 기능, JavaScript 라이브러리, 상세 예제는 [reference.md](reference.md)를 참고한다. PDF 양식을 채워야 하는 경우 [forms.md](forms.md)를 읽고 그 지침을 따른다.

## 빠른 시작

```python
from pypdf import PdfReader, PdfWriter

# PDF 읽기
reader = PdfReader("document.pdf")
print(f"Pages: {len(reader.pages)}")

# 텍스트 추출
text = ""
for page in reader.pages:
    text += page.extract_text()
```

## Python 라이브러리

### pypdf - 기본 작업

#### PDF 합치기
```python
from pypdf import PdfWriter, PdfReader

writer = PdfWriter()
for pdf_file in ["doc1.pdf", "doc2.pdf", "doc3.pdf"]:
    reader = PdfReader(pdf_file)
    for page in reader.pages:
        writer.add_page(page)

with open("merged.pdf", "wb") as output:
    writer.write(output)
```

#### PDF 분할
```python
reader = PdfReader("input.pdf")
for i, page in enumerate(reader.pages):
    writer = PdfWriter()
    writer.add_page(page)
    with open(f"page_{i+1}.pdf", "wb") as output:
        writer.write(output)
```

#### 메타데이터 추출
```python
reader = PdfReader("document.pdf")
meta = reader.metadata
print(f"Title: {meta.title}")
print(f"Author: {meta.author}")
print(f"Subject: {meta.subject}")
print(f"Creator: {meta.creator}")
```

#### 페이지 회전
```python
reader = PdfReader("input.pdf")
writer = PdfWriter()

page = reader.pages[0]
page.rotate(90)  # 시계 방향 90도 회전
writer.add_page(page)

with open("rotated.pdf", "wb") as output:
    writer.write(output)
```

### pdfplumber - 텍스트 및 표 추출

#### 레이아웃 보존 텍스트 추출
```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    for page in pdf.pages:
        text = page.extract_text()
        print(text)
```

#### 표 추출
```python
with pdfplumber.open("document.pdf") as pdf:
    for i, page in enumerate(pdf.pages):
        tables = page.extract_tables()
        for j, table in enumerate(tables):
            print(f"Table {j+1} on page {i+1}:")
            for row in table:
                print(row)
```

#### 고급 표 추출
```python
import pandas as pd

with pdfplumber.open("document.pdf") as pdf:
    all_tables = []
    for page in pdf.pages:
        tables = page.extract_tables()
        for table in tables:
            if table:
                df = pd.DataFrame(table[1:], columns=table[0])
                all_tables.append(df)

if all_tables:
    combined_df = pd.concat(all_tables, ignore_index=True)
    combined_df.to_excel("extracted_tables.xlsx", index=False)
```

### reportlab - PDF 생성

#### 기본 PDF 생성
```python
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas

c = canvas.Canvas("hello.pdf", pagesize=letter)
width, height = letter

c.drawString(100, height - 100, "Hello World!")
c.drawString(100, height - 120, "This is a PDF created with reportlab")
c.line(100, height - 140, 400, height - 140)
c.save()
```

#### 여러 페이지 PDF 생성
```python
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, PageBreak
from reportlab.lib.styles import getSampleStyleSheet

doc = SimpleDocTemplate("report.pdf", pagesize=letter)
styles = getSampleStyleSheet()
story = []

title = Paragraph("Report Title", styles['Title'])
story.append(title)
story.append(Spacer(1, 12))

body = Paragraph("This is the body of the report. " * 20, styles['Normal'])
story.append(body)
story.append(PageBreak())

story.append(Paragraph("Page 2", styles['Heading1']))
story.append(Paragraph("Content for page 2", styles['Normal']))

doc.build(story)
```

#### 위첨자/아래첨자

**중요**: ReportLab PDF에서 유니코드 위첨자/아래첨자 문자(₀₁₂₃⁰¹²³ 등)를 절대 사용하지 않는다. 내장 폰트에 해당 글리프가 없어 검은 사각형으로 렌더링된다.

대신 Paragraph 객체에서 ReportLab XML 마크업 태그를 사용한다:
```python
from reportlab.platypus import Paragraph
from reportlab.lib.styles import getSampleStyleSheet

styles = getSampleStyleSheet()

# 아래첨자: <sub> 태그 사용
chemical = Paragraph("H<sub>2</sub>O", styles['Normal'])

# 위첨자: <super> 태그 사용
squared = Paragraph("x<super>2</super> + y<super>2</super>", styles['Normal'])
```

canvas로 텍스트를 직접 그리는 경우에는 폰트 크기와 위치를 수동으로 조정한다.

## 커맨드라인 도구

### pdftotext (poppler-utils)
```bash
# 텍스트 추출
pdftotext input.pdf output.txt

# 레이아웃 보존 텍스트 추출
pdftotext -layout input.pdf output.txt

# 특정 페이지 추출
pdftotext -f 1 -l 5 input.pdf output.txt  # 1~5페이지
```

### qpdf
```bash
# PDF 합치기
qpdf --empty --pages file1.pdf file2.pdf -- merged.pdf

# 페이지 분할
qpdf input.pdf --pages . 1-5 -- pages1-5.pdf
qpdf input.pdf --pages . 6-10 -- pages6-10.pdf

# 페이지 회전
qpdf input.pdf output.pdf --rotate=+90:1  # 1페이지 90도 회전

# 비밀번호 제거
qpdf --password=mypassword --decrypt encrypted.pdf decrypted.pdf
```

### pdftk (설치된 경우)
```bash
# 합치기
pdftk file1.pdf file2.pdf cat output merged.pdf

# 분할
pdftk input.pdf burst

# 회전
pdftk input.pdf rotate 1east output rotated.pdf
```

## 주요 작업

### 스캔 PDF에서 텍스트 추출
```python
# 필요: pip install pytesseract pdf2image
import pytesseract
from pdf2image import convert_from_path

images = convert_from_path('scanned.pdf')

text = ""
for i, image in enumerate(images):
    text += f"Page {i+1}:\n"
    text += pytesseract.image_to_string(image)
    text += "\n\n"

print(text)
```

### 워터마크 추가
```python
from pypdf import PdfReader, PdfWriter

watermark = PdfReader("watermark.pdf").pages[0]

reader = PdfReader("document.pdf")
writer = PdfWriter()

for page in reader.pages:
    page.merge_page(watermark)
    writer.add_page(page)

with open("watermarked.pdf", "wb") as output:
    writer.write(output)
```

### 이미지 추출
```bash
# pdfimages (poppler-utils) 사용
pdfimages -j input.pdf output_prefix
# output_prefix-000.jpg, output_prefix-001.jpg 등으로 추출됨
```

### 비밀번호 보호
```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("input.pdf")
writer = PdfWriter()

for page in reader.pages:
    writer.add_page(page)

writer.encrypt("userpassword", "ownerpassword")

with open("encrypted.pdf", "wb") as output:
    writer.write(output)
```

## 빠른 참조

| 작업 | 최적 도구 | 명령/코드 |
|------|-----------|-----------|
| PDF 합치기 | pypdf | `writer.add_page(page)` |
| PDF 분할 | pypdf | 페이지별 파일 생성 |
| 텍스트 추출 | pdfplumber | `page.extract_text()` |
| 표 추출 | pdfplumber | `page.extract_tables()` |
| PDF 생성 | reportlab | Canvas 또는 Platypus |
| 커맨드라인 합치기 | qpdf | `qpdf --empty --pages ...` |
| 스캔 PDF OCR | pytesseract | 먼저 이미지로 변환 |
| PDF 양식 채우기 | pdf-lib 또는 pypdf (forms.md 참고) | forms.md 참고 |

## 다음 단계

- 고급 pypdfium2 사용법은 [reference.md](reference.md) 참고
- JavaScript 라이브러리(pdf-lib)는 [reference.md](reference.md) 참고
- PDF 양식 채우기는 [forms.md](forms.md)의 지침을 따를 것
- 문제 해결 가이드는 [reference.md](reference.md) 참고
