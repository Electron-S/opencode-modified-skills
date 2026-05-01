# PDF 처리 고급 참조

이 문서는 메인 스킬에서 다루지 않는 고급 PDF 처리 기능, 상세 예제, 추가 라이브러리를 다룬다.

## pypdfium2 라이브러리 (Apache/BSD 라이선스)

### 개요
pypdfium2는 PDFium(크롬의 PDF 라이브러리)용 Python 바인딩이다. 빠른 PDF 렌더링, 이미지 생성에 적합하며 PyMuPDF의 대체재로 사용된다.

### PDF를 이미지로 렌더링
```python
import pypdfium2 as pdfium
from PIL import Image

pdf = pdfium.PdfDocument("document.pdf")

page = pdf[0]  # 첫 번째 페이지
bitmap = page.render(
    scale=2.0,  # 고해상도
    rotation=0
)

img = bitmap.to_pil()
img.save("page_1.png", "PNG")

for i, page in enumerate(pdf):
    bitmap = page.render(scale=1.5)
    img = bitmap.to_pil()
    img.save(f"page_{i+1}.jpg", "JPEG", quality=90)
```

### pypdfium2로 텍스트 추출
```python
import pypdfium2 as pdfium

pdf = pdfium.PdfDocument("document.pdf")
for i, page in enumerate(pdf):
    text = page.get_text()
    print(f"Page {i+1} text length: {len(text)} chars")
```

## JavaScript 라이브러리

### pdf-lib (MIT 라이선스)

pdf-lib은 모든 JavaScript 환경에서 PDF 문서를 생성하고 수정할 수 있는 강력한 라이브러리이다.

#### 기존 PDF 로드 및 조작
```javascript
import { PDFDocument } from 'pdf-lib';
import fs from 'fs';

async function manipulatePDF() {
    const existingPdfBytes = fs.readFileSync('input.pdf');
    const pdfDoc = await PDFDocument.load(existingPdfBytes);

    const pageCount = pdfDoc.getPageCount();
    console.log(`Document has ${pageCount} pages`);

    const newPage = pdfDoc.addPage([600, 400]);
    newPage.drawText('Added by pdf-lib', {
        x: 100,
        y: 300,
        size: 16
    });

    const pdfBytes = await pdfDoc.save();
    fs.writeFileSync('modified.pdf', pdfBytes);
}
```

#### 처음부터 복잡한 PDF 생성
```javascript
import { PDFDocument, rgb, StandardFonts } from 'pdf-lib';
import fs from 'fs';

async function createPDF() {
    const pdfDoc = await PDFDocument.create();

    const helveticaFont = await pdfDoc.embedFont(StandardFonts.Helvetica);
    const helveticaBold = await pdfDoc.embedFont(StandardFonts.HelveticaBold);

    const page = pdfDoc.addPage([595, 842]); // A4 크기
    const { width, height } = page.getSize();

    page.drawText('Invoice #12345', {
        x: 50,
        y: height - 50,
        size: 18,
        font: helveticaBold,
        color: rgb(0.2, 0.2, 0.8)
    });

    page.drawRectangle({
        x: 40,
        y: height - 100,
        width: width - 80,
        height: 30,
        color: rgb(0.9, 0.9, 0.9)
    });

    const items = [
        ['Item', 'Qty', 'Price', 'Total'],
        ['Widget', '2', '$50', '$100'],
        ['Gadget', '1', '$75', '$75']
    ];

    let yPos = height - 150;
    items.forEach(row => {
        let xPos = 50;
        row.forEach(cell => {
            page.drawText(cell, { x: xPos, y: yPos, size: 12, font: helveticaFont });
            xPos += 120;
        });
        yPos -= 25;
    });

    const pdfBytes = await pdfDoc.save();
    fs.writeFileSync('created.pdf', pdfBytes);
}
```

#### 고급 합치기 및 분할
```javascript
import { PDFDocument } from 'pdf-lib';
import fs from 'fs';

async function mergePDFs() {
    const mergedPdf = await PDFDocument.create();

    const pdf1Bytes = fs.readFileSync('doc1.pdf');
    const pdf2Bytes = fs.readFileSync('doc2.pdf');

    const pdf1 = await PDFDocument.load(pdf1Bytes);
    const pdf2 = await PDFDocument.load(pdf2Bytes);

    const pdf1Pages = await mergedPdf.copyPages(pdf1, pdf1.getPageIndices());
    pdf1Pages.forEach(page => mergedPdf.addPage(page));

    // 두 번째 PDF에서 특정 페이지만 복사 (0, 2, 4 페이지)
    const pdf2Pages = await mergedPdf.copyPages(pdf2, [0, 2, 4]);
    pdf2Pages.forEach(page => mergedPdf.addPage(page));

    const mergedPdfBytes = await mergedPdf.save();
    fs.writeFileSync('merged.pdf', mergedPdfBytes);
}
```

### pdfjs-dist (Apache 라이선스)

PDF.js는 브라우저에서 PDF를 렌더링하기 위한 Mozilla의 JavaScript 라이브러리이다.

#### 기본 PDF 로드 및 렌더링
```javascript
import * as pdfjsLib from 'pdfjs-dist';

pdfjsLib.GlobalWorkerOptions.workerSrc = './pdf.worker.js';

async function renderPDF() {
    const loadingTask = pdfjsLib.getDocument('document.pdf');
    const pdf = await loadingTask.promise;

    console.log(`Loaded PDF with ${pdf.numPages} pages`);

    const page = await pdf.getPage(1);
    const viewport = page.getViewport({ scale: 1.5 });

    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');
    canvas.height = viewport.height;
    canvas.width = viewport.width;

    await page.render({ canvasContext: context, viewport }).promise;
    document.body.appendChild(canvas);
}
```

#### 좌표와 함께 텍스트 추출
```javascript
import * as pdfjsLib from 'pdfjs-dist';

async function extractText() {
    const loadingTask = pdfjsLib.getDocument('document.pdf');
    const pdf = await loadingTask.promise;

    let fullText = '';

    for (let i = 1; i <= pdf.numPages; i++) {
        const page = await pdf.getPage(i);
        const textContent = await page.getTextContent();

        const pageText = textContent.items.map(item => item.str).join(' ');
        fullText += `\n--- Page ${i} ---\n${pageText}`;

        const textWithCoords = textContent.items.map(item => ({
            text: item.str,
            x: item.transform[4],
            y: item.transform[5],
            width: item.width,
            height: item.height
        }));
    }

    console.log(fullText);
    return fullText;
}
```

## 고급 커맨드라인 작업

### poppler-utils 고급 기능

#### 바운딩 박스 좌표와 함께 텍스트 추출
```bash
# 바운딩 박스 좌표와 함께 텍스트 추출 (구조적 데이터에 필수)
pdftotext -bbox-layout document.pdf output.xml
```

#### 고급 이미지 변환
```bash
# 특정 해상도로 PNG 이미지로 변환
pdftoppm -png -r 300 document.pdf output_prefix

# 특정 페이지 범위를 고해상도로 변환
pdftoppm -png -r 600 -f 1 -l 3 document.pdf high_res_pages

# 품질 설정과 함께 JPEG로 변환
pdftoppm -jpeg -jpegopt quality=85 -r 200 document.pdf jpeg_output
```

#### 내장 이미지 추출
```bash
# 메타데이터와 함께 모든 이미지 추출
pdfimages -j -p document.pdf page_images

# 추출 없이 이미지 정보 목록 표시
pdfimages -list document.pdf

# 원본 형식으로 이미지 추출
pdfimages -all document.pdf images/img
```

### qpdf 고급 기능

#### 복잡한 페이지 조작
```bash
# 페이지 그룹으로 PDF 분할
qpdf --split-pages=3 input.pdf output_group_%02d.pdf

# 복잡한 범위로 특정 페이지 추출
qpdf input.pdf --pages input.pdf 1,3-5,8,10-end -- extracted.pdf

# 여러 PDF에서 특정 페이지 합치기
qpdf --empty --pages doc1.pdf 1-3 doc2.pdf 5-7 doc3.pdf 2,4 -- combined.pdf
```

#### PDF 최적화 및 복구
```bash
# 웹용 PDF 최적화 (스트리밍을 위한 선형화)
qpdf --linearize input.pdf optimized.pdf

# 사용하지 않는 객체 제거 및 압축
qpdf --optimize-level=all input.pdf compressed.pdf

# 손상된 PDF 구조 복구 시도
qpdf --check input.pdf
qpdf --fix-qdf damaged.pdf repaired.pdf
```

#### 고급 암호화
```bash
# 특정 권한으로 비밀번호 보호 추가
qpdf --encrypt user_pass owner_pass 256 --print=none --modify=none -- input.pdf encrypted.pdf

# 암호화 상태 확인
qpdf --show-encryption encrypted.pdf
```

## 고급 Python 기법

### pdfplumber 고급 기능

#### 정확한 좌표와 함께 텍스트 추출
```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    page = pdf.pages[0]

    chars = page.chars
    for char in chars[:10]:
        print(f"Char: '{char['text']}' at x:{char['x0']:.1f} y:{char['y0']:.1f}")

    # 바운딩 박스로 텍스트 추출 (left, top, right, bottom)
    bbox_text = page.within_bbox((100, 100, 400, 200)).extract_text()
```

#### 사용자 설정을 이용한 고급 표 추출
```python
import pdfplumber
import pandas as pd

with pdfplumber.open("complex_table.pdf") as pdf:
    page = pdf.pages[0]

    table_settings = {
        "vertical_strategy": "lines",
        "horizontal_strategy": "lines",
        "snap_tolerance": 3,
        "intersection_tolerance": 15
    }
    tables = page.extract_tables(table_settings)

    img = page.to_image(resolution=150)
    img.save("debug_layout.png")
```

### reportlab 고급 기능

#### 표가 있는 전문 보고서 생성
```python
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib import colors

data = [
    ['Product', 'Q1', 'Q2', 'Q3', 'Q4'],
    ['Widgets', '120', '135', '142', '158'],
    ['Gadgets', '85', '92', '98', '105']
]

doc = SimpleDocTemplate("report.pdf")
elements = []

styles = getSampleStyleSheet()
title = Paragraph("Quarterly Sales Report", styles['Title'])
elements.append(title)

table = Table(data)
table.setStyle(TableStyle([
    ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
    ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
    ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
    ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
    ('FONTSIZE', (0, 0), (-1, 0), 14),
    ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
    ('BACKGROUND', (0, 1), (-1, -1), colors.beige),
    ('GRID', (0, 0), (-1, -1), 1, colors.black)
]))
elements.append(table)

doc.build(elements)
```

## 복잡한 워크플로

### PDF에서 이미지 추출

#### 방법 1: pdfimages 사용 (가장 빠름)
```bash
pdfimages -all document.pdf images/img
```

#### 방법 2: pypdfium2 + 이미지 처리
```python
import pypdfium2 as pdfium
from PIL import Image
import numpy as np

def extract_figures(pdf_path, output_dir):
    pdf = pdfium.PdfDocument(pdf_path)

    for page_num, page in enumerate(pdf):
        bitmap = page.render(scale=3.0)
        img = bitmap.to_pil()
        img_array = np.array(img)
        mask = np.any(img_array != [255, 255, 255], axis=2)
        # 실제 구현은 더 정교한 감지 로직 필요
```

### 오류 처리를 포함한 배치 PDF 처리
```python
import os
import glob
from pypdf import PdfReader, PdfWriter
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def batch_process_pdfs(input_dir, operation='merge'):
    pdf_files = glob.glob(os.path.join(input_dir, "*.pdf"))

    if operation == 'merge':
        writer = PdfWriter()
        for pdf_file in pdf_files:
            try:
                reader = PdfReader(pdf_file)
                for page in reader.pages:
                    writer.add_page(page)
                logger.info(f"Processed: {pdf_file}")
            except Exception as e:
                logger.error(f"Failed to process {pdf_file}: {e}")
                continue

        with open("batch_merged.pdf", "wb") as output:
            writer.write(output)

    elif operation == 'extract_text':
        for pdf_file in pdf_files:
            try:
                reader = PdfReader(pdf_file)
                text = ""
                for page in reader.pages:
                    text += page.extract_text()

                output_file = pdf_file.replace('.pdf', '.txt')
                with open(output_file, 'w', encoding='utf-8') as f:
                    f.write(text)
                logger.info(f"Extracted text from: {pdf_file}")

            except Exception as e:
                logger.error(f"Failed to extract text from {pdf_file}: {e}")
                continue
```

### 고급 PDF 자르기
```python
from pypdf import PdfWriter, PdfReader

reader = PdfReader("input.pdf")
writer = PdfWriter()

page = reader.pages[0]
page.mediabox.left = 50
page.mediabox.bottom = 50
page.mediabox.right = 550
page.mediabox.top = 750

writer.add_page(page)
with open("cropped.pdf", "wb") as output:
    writer.write(output)
```

## 성능 최적화 팁

### 1. 대용량 PDF
- 전체 PDF를 메모리에 로드하는 대신 스트리밍 방식 사용
- `qpdf --split-pages`로 대용량 파일 분할
- pypdfium2로 페이지를 개별 처리

### 2. 텍스트 추출
- `pdftotext -bbox-layout`이 일반 텍스트 추출에 가장 빠름
- 구조적 데이터와 표에는 pdfplumber 사용
- 매우 큰 문서에서 `pypdf.extract_text()` 사용 지양

### 3. 이미지 추출
- `pdfimages`가 페이지 렌더링보다 훨씬 빠름
- 미리보기에는 저해상도, 최종 출력에는 고해상도 사용

### 4. 메모리 관리
```python
def process_large_pdf(pdf_path, chunk_size=10):
    reader = PdfReader(pdf_path)
    total_pages = len(reader.pages)

    for start_idx in range(0, total_pages, chunk_size):
        end_idx = min(start_idx + chunk_size, total_pages)
        writer = PdfWriter()

        for i in range(start_idx, end_idx):
            writer.add_page(reader.pages[i])

        with open(f"chunk_{start_idx//chunk_size}.pdf", "wb") as output:
            writer.write(output)
```

## 일반적인 문제 해결

### 암호화된 PDF
```python
from pypdf import PdfReader

try:
    reader = PdfReader("encrypted.pdf")
    if reader.is_encrypted:
        reader.decrypt("password")
except Exception as e:
    print(f"Failed to decrypt: {e}")
```

### 손상된 PDF
```bash
qpdf --check corrupted.pdf
qpdf --replace-input corrupted.pdf
```

### 텍스트 추출 문제
```python
# 스캔 PDF의 경우 OCR 대체
import pytesseract
from pdf2image import convert_from_path

def extract_text_with_ocr(pdf_path):
    images = convert_from_path(pdf_path)
    text = ""
    for i, image in enumerate(images):
        text += pytesseract.image_to_string(image)
    return text
```

## 라이선스 정보

- **pypdf**: BSD License
- **pdfplumber**: MIT License
- **pypdfium2**: Apache/BSD License
- **reportlab**: BSD License
- **poppler-utils**: GPL-2 License
- **qpdf**: Apache License
- **pdf-lib**: MIT License
- **pdfjs-dist**: Apache License
