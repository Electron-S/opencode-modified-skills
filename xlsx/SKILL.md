---
name: xlsx
description: 스프레드시트 파일이 주요 입력 또는 출력인 모든 작업에 이 스킬을 사용한다. 기존 .xlsx, .xlsm, .csv, .tsv 파일을 열거나 읽거나 편집하거나 수정하는 작업(열 추가, 수식 계산, 서식 지정, 차트 작성, 데이터 정리 등), 처음부터 새 스프레드시트를 만들거나 다른 데이터 소스에서 생성하는 작업, 표 형식 파일 간 변환 작업이 모두 해당된다. 사용자가 스프레드시트 파일을 이름이나 경로로 언급하면서 무언가를 요청할 때도 사용한다. 지저분한 표 형식 데이터(잘못된 행, 잘못된 헤더, 불량 데이터)를 정리하거나 재구성하는 경우도 해당된다. 결과물이 스프레드시트 파일이어야 한다. Word 문서, HTML 보고서, 단독 Python 스크립트, 데이터베이스 파이프라인, Google Sheets API 연동이 주 결과물인 경우에는 사용하지 않는다.
---

# 출력물 요구사항

## 모든 Excel 파일

### 전문적인 폰트
- 별도 지시가 없으면 모든 결과물에 일관된 전문 폰트(Arial, Times New Roman 등) 사용

### 수식 오류 제로
- 모든 Excel 모델은 수식 오류(#REF!, #DIV/0!, #VALUE!, #N/A, #NAME?)가 없어야 한다

### 기존 템플릿 보존 (템플릿 수정 시)
- 파일 수정 시 기존 형식, 스타일, 관례를 정확히 파악하고 그대로 따른다
- 기존 패턴이 있는 파일에 표준화된 서식을 강요하지 않는다
- 기존 템플릿 관례가 항상 이 가이드라인보다 우선한다

## 재무 모델

### 색상 코딩 표준
사용자나 기존 템플릿이 달리 지정하지 않으면 아래 업계 표준 색상을 따른다

#### 업계 표준 색상 관례
- **파란색 텍스트 (RGB: 0,0,255)**: 하드코딩된 입력값, 시나리오에서 사용자가 변경할 숫자
- **검은색 텍스트 (RGB: 0,0,0)**: 모든 수식 및 계산
- **초록색 텍스트 (RGB: 0,128,0)**: 같은 워크북 내 다른 워크시트에서 가져오는 링크
- **빨간색 텍스트 (RGB: 255,0,0)**: 다른 파일에 대한 외부 링크
- **노란색 배경 (RGB: 255,255,0)**: 주의가 필요한 핵심 가정 또는 업데이트가 필요한 셀

### 숫자 서식 표준

#### 필수 서식 규칙
- **연도**: 텍스트 문자열로 서식 지정 (예: "2024", "2,024"가 아님)
- **통화**: $#,##0 서식 사용; 헤더에 항상 단위 명시 ("Revenue ($mm)")
- **0**: 숫자 서식으로 모든 0을 "-"로 표시, 백분율 포함 (예: "$#,##0;($#,##0);-")
- **백분율**: 기본 0.0% 서식 (소수점 한 자리)
- **배수**: 밸류에이션 배수에 0.0x 서식 사용 (EV/EBITDA, P/E)
- **음수**: 괄호 사용 (123), 마이너스 기호(-123) 사용 안 함

### 수식 작성 규칙

#### 가정 배치
- 모든 가정(성장률, 마진, 배수 등)은 별도 가정 셀에 배치
- 수식에서 하드코딩 값 대신 셀 참조 사용
- 예시: =B5*1.05 대신 =B5*(1+$B$6) 사용

#### 수식 오류 예방
- 모든 셀 참조가 올바른지 확인
- 범위의 off-by-one 오류 확인
- 모든 예측 기간에 걸쳐 일관된 수식 유지
- 엣지 케이스(0값, 음수, 매우 큰 값) 테스트
- 의도치 않은 순환 참조 없는지 확인

#### 하드코드 문서화 요구사항
- 셀 옆 또는 표 끝에 주석 추가. 형식: "Source: [시스템/문서], [날짜], [구체적 참조], [URL]"
- 예시:
  - "Source: Company 10-K, FY2024, Page 45, Revenue Note"
  - "Source: Bloomberg Terminal, 8/15/2025, AAPL US Equity"

---

# XLSX 생성, 편집, 분석

## 개요

사용자가 .xlsx 파일을 생성, 편집, 분석하도록 요청할 수 있다. 작업에 따라 다양한 도구와 워크플로를 사용할 수 있다.

## 중요 요구사항

**수식 재계산에 LibreOffice 필요**: `~/.config/opencode/skills/xlsx/scripts/recalc.py` 스크립트로 수식 값을 재계산한다. LibreOffice가 설치되어 있다고 가정한다.

## 데이터 읽기 및 분석

### pandas로 데이터 분석
데이터 분석, 시각화, 기본 작업에는 **pandas** 사용:

```python
import pandas as pd

# Excel 읽기
df = pd.read_excel('file.xlsx')
all_sheets = pd.read_excel('file.xlsx', sheet_name=None)  # 모든 시트

# 분석
df.head()
df.info()
df.describe()

# Excel 쓰기
df.to_excel('output.xlsx', index=False)
```

## Excel 파일 워크플로

## 중요: 하드코딩 값이 아닌 수식 사용

**Python에서 값을 계산하여 하드코딩하지 않고, 항상 Excel 수식을 사용한다.** 스프레드시트가 동적이고 업데이트 가능한 상태를 유지하도록 한다.

### ❌ 잘못된 방법 - 하드코딩된 계산 값
```python
# 나쁜 예: Python에서 계산하여 결과를 하드코딩
total = df['Sales'].sum()
sheet['B10'] = total  # 5000을 하드코딩

# 나쁜 예: Python에서 성장률 계산
growth = (df.iloc[-1]['Revenue'] - df.iloc[0]['Revenue']) / df.iloc[0]['Revenue']
sheet['C5'] = growth  # 0.15를 하드코딩
```

### ✅ 올바른 방법 - Excel 수식 사용
```python
# 좋은 예: Excel이 합계를 계산하게 함
sheet['B10'] = '=SUM(B2:B9)'

# 좋은 예: Excel 수식으로 성장률 계산
sheet['C5'] = '=(C4-C2)/C2'

# 좋은 예: Excel 함수로 평균 계산
sheet['D20'] = '=AVERAGE(D2:D19)'
```

이는 모든 계산(합계, 백분율, 비율, 차이 등)에 적용된다.

## 일반 워크플로
1. **도구 선택**: 데이터에는 pandas, 수식/서식에는 openpyxl
2. **생성/로드**: 새 워크북 생성 또는 기존 파일 로드
3. **수정**: 데이터, 수식, 서식 추가/편집
4. **저장**: 파일에 저장
5. **수식 재계산 (수식 사용 시 필수)**:
   ```bash
   python ~/.config/opencode/skills/xlsx/scripts/recalc.py output.xlsx
   ```
6. **오류 확인 및 수정**:
   - 스크립트가 오류 세부사항이 있는 JSON 반환
   - `status`가 `errors_found`이면 `error_summary`에서 구체적인 오류 유형과 위치 확인
   - 오류 수정 후 다시 재계산
   - 일반적인 오류: `#REF!`(잘못된 셀 참조), `#DIV/0!`(0으로 나누기), `#VALUE!`(잘못된 데이터 타입), `#NAME?`(인식되지 않는 수식 이름)

### 새 Excel 파일 생성

```python
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment

wb = Workbook()
sheet = wb.active

sheet['A1'] = 'Hello'
sheet['B1'] = 'World'
sheet.append(['Row', 'of', 'data'])

# 수식 추가
sheet['B2'] = '=SUM(A1:A10)'

# 서식
sheet['A1'].font = Font(bold=True, color='FF0000')
sheet['A1'].fill = PatternFill('solid', start_color='FFFF00')
sheet['A1'].alignment = Alignment(horizontal='center')

sheet.column_dimensions['A'].width = 20

wb.save('output.xlsx')
```

### 기존 Excel 파일 편집

```python
from openpyxl import load_workbook

wb = load_workbook('existing.xlsx')
sheet = wb.active

for sheet_name in wb.sheetnames:
    sheet = wb[sheet_name]
    print(f"Sheet: {sheet_name}")

sheet['A1'] = 'New Value'
sheet.insert_rows(2)
sheet.delete_cols(3)

new_sheet = wb.create_sheet('NewSheet')
new_sheet['A1'] = 'Data'

wb.save('modified.xlsx')
```

## 수식 재계산

openpyxl로 생성하거나 수정한 Excel 파일은 수식이 문자열로만 저장된다. 제공된 스크립트로 수식을 재계산한다:

```bash
python ~/.config/opencode/skills/xlsx/scripts/recalc.py <excel_file> [timeout_seconds]
```

예시:
```bash
python ~/.config/opencode/skills/xlsx/scripts/recalc.py output.xlsx 30
```

스크립트 기능:
- 첫 실행 시 LibreOffice 매크로 자동 설정
- 모든 시트의 모든 수식 재계산
- 모든 셀의 Excel 오류 스캔 (#REF!, #DIV/0! 등)
- 오류 위치와 개수가 포함된 JSON 반환

## 수식 검증 체크리스트

### 필수 검증
- [ ] **샘플 참조 2~3개 테스트**: 전체 모델 구축 전에 올바른 값을 가져오는지 확인
- [ ] **열 매핑**: Excel 열이 일치하는지 확인 (예: 64열 = BL, BK가 아님)
- [ ] **행 오프셋**: Excel 행은 1부터 시작 (DataFrame 5행 = Excel 6행)

### 일반적인 실수
- [ ] **NaN 처리**: `pd.notna()`로 null 값 확인
- [ ] **오른쪽 끝 열**: 회계연도 데이터는 종종 50번 이상의 열에 있음
- [ ] **다중 일치**: 첫 번째만 아닌 모든 항목 검색
- [ ] **0으로 나누기**: 수식에서 `/` 사용 전 분모 확인 (#DIV/0!)
- [ ] **잘못된 참조**: 모든 셀 참조가 의도한 셀을 가리키는지 확인 (#REF!)
- [ ] **시트 간 참조**: 시트 연결에 올바른 형식 사용 (Sheet1!A1)

### scripts/recalc.py 출력 해석
```json
{
  "status": "success",
  "total_errors": 0,
  "total_formulas": 42,
  "error_summary": {
    "#REF!": {
      "count": 2,
      "locations": ["Sheet1!B5", "Sheet1!C10"]
    }
  }
}
```

## 모범 사례

### 라이브러리 선택
- **pandas**: 데이터 분석, 대량 작업, 단순 데이터 내보내기에 적합
- **openpyxl**: 복잡한 서식, 수식, Excel 전용 기능에 적합

### openpyxl 사용 시
- 셀 인덱스는 1부터 시작 (row=1, column=1은 A1 셀)
- 계산된 값을 읽으려면 `data_only=True` 사용: `load_workbook('file.xlsx', data_only=True)`
- **경고**: `data_only=True`로 열고 저장하면 수식이 값으로 영구 교체됨
- 대용량 파일: 읽기에는 `read_only=True`, 쓰기에는 `write_only=True` 사용

### pandas 사용 시
- 데이터 타입 명시: `pd.read_excel('file.xlsx', dtype={'id': str})`
- 대용량 파일에서 특정 열만 읽기: `pd.read_excel('file.xlsx', usecols=['A', 'C', 'E'])`
- 날짜 올바르게 처리: `pd.read_excel('file.xlsx', parse_dates=['date_column'])`

## 코드 스타일 가이드라인
- Python 코드는 불필요한 주석 없이 간결하게 작성
- 불필요한 print 문 사용 지양
- Excel 파일 자체에는 복잡한 수식이나 중요한 가정에 셀 주석 추가
