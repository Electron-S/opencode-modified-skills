**중요: 아래 단계를 순서대로 완료해야 한다. 코드 작성으로 건너뛰지 않는다.**

PDF 양식을 채워야 하는 경우, 먼저 PDF에 채울 수 있는 양식 필드가 있는지 확인한다. 다음 스크립트를 실행한다:

```bash
python ~/.config/opencode/skills/pdf/scripts/check_fillable_fields.py <file.pdf>
```

결과에 따라 "채울 수 있는 필드" 또는 "채울 수 없는 필드" 섹션으로 이동한다.

---

# 채울 수 있는 필드

PDF에 채울 수 있는 양식 필드가 있는 경우:

- 다음 스크립트로 필드 정보 JSON 파일을 생성한다:
  ```bash
  python ~/.config/opencode/skills/pdf/scripts/extract_form_field_info.py <input.pdf> <field_info.json>
  ```
  생성되는 JSON 형식:
  ```json
  [
    {
      "field_id": "(필드 고유 ID)",
      "page": "(페이지 번호, 1부터 시작)",
      "rect": "[left, bottom, right, top] PDF 좌표 바운딩 박스, y=0은 페이지 하단",
      "type": "text, checkbox, radio_group, choice 중 하나"
    },
    {
      "field_id": "(고유 ID)",
      "page": 1,
      "type": "checkbox",
      "checked_value": "(체크박스를 선택하려면 이 값으로 설정)",
      "unchecked_value": "(체크박스를 해제하려면 이 값으로 설정)"
    },
    {
      "field_id": "(고유 ID)",
      "page": 1,
      "type": "radio_group",
      "radio_options": [
        {
          "value": "(이 값으로 설정하면 이 라디오 옵션이 선택됨)",
          "rect": "(해당 라디오 버튼의 바운딩 박스)"
        }
      ]
    },
    {
      "field_id": "(고유 ID)",
      "page": 1,
      "type": "choice",
      "choice_options": [
        {
          "value": "(이 값으로 설정하면 이 옵션이 선택됨)",
          "text": "(옵션의 표시 텍스트)"
        }
      ]
    }
  ]
  ```

- 다음 스크립트로 PDF를 PNG 이미지(페이지별 한 장)로 변환한다:
  ```bash
  python ~/.config/opencode/skills/pdf/scripts/convert_pdf_to_images.py <file.pdf> <output_directory>
  ```
  이미지를 분석하여 각 양식 필드의 용도를 파악한다(바운딩 박스 PDF 좌표를 이미지 좌표로 변환하는 것을 잊지 않는다).

- 각 필드에 입력할 값을 담은 `field_values.json` 파일을 다음 형식으로 작성한다:
  ```json
  [
    {
      "field_id": "last_name",
      "description": "사용자 성(last name)",
      "page": 1,
      "value": "Simpson"
    },
    {
      "field_id": "Checkbox12",
      "description": "18세 이상인 경우 체크",
      "page": 1,
      "value": "/On"
    }
  ]
  ```

- 다음 스크립트로 채워진 PDF를 생성한다:
  ```bash
  python ~/.config/opencode/skills/pdf/scripts/fill_fillable_fields.py <input.pdf> <field_values.json> <output.pdf>
  ```
  스크립트가 필드 ID와 값의 유효성을 검사한다. 오류 메시지가 출력되면 해당 필드를 수정하고 다시 시도한다.

---

# 채울 수 없는 필드

PDF에 채울 수 있는 양식 필드가 없는 경우, 텍스트 주석을 추가한다. 먼저 PDF 구조에서 좌표를 추출하고(더 정확), 실패 시 시각적 추정으로 대체한다.

## 1단계: 구조 추출 시도

다음 스크립트로 텍스트 레이블, 선, 체크박스와 정확한 PDF 좌표를 추출한다:
```bash
python ~/.config/opencode/skills/pdf/scripts/extract_form_structure.py <input.pdf> form_structure.json
```

생성되는 JSON에는 다음이 포함된다:
- **labels**: 정확한 좌표가 있는 모든 텍스트 요소 (x0, top, x1, bottom, PDF 포인트 단위)
- **lines**: 행 경계를 정의하는 수평선
- **checkboxes**: 체크박스인 작은 사각형 (중심 좌표 포함)
- **row_boundaries**: 수평선에서 계산된 행 상단/하단 위치

**결과 확인**: `form_structure.json`에 양식 필드에 해당하는 텍스트 레이블이 있으면 **A 방법(구조 기반 좌표)**을 사용한다. PDF가 스캔/이미지 기반이고 레이블이 거의 없으면 **B 방법(시각적 추정)**을 사용한다.

---

## A 방법: 구조 기반 좌표 (권장)

`extract_form_structure.py`에서 텍스트 레이블을 찾은 경우에 사용한다.

### A.1: 구조 분석

form_structure.json을 읽고 다음을 파악한다:

1. **레이블 그룹**: 단일 레이블을 구성하는 인접 텍스트 요소 (예: "Last" + "Name")
2. **행 구조**: `top` 값이 비슷한 레이블은 같은 행
3. **필드 열**: 입력 영역은 레이블 끝 이후에 시작 (x0 = label.x1 + gap)
4. **체크박스**: 구조에서 직접 체크박스 좌표 사용

**좌표 시스템**: PDF 좌표에서 y=0은 페이지 상단, y는 아래로 증가한다.

### A.2: 누락 요소 확인

구조 추출이 모든 양식 요소를 감지하지 못할 수 있다:
- **원형 체크박스**: 사각형만 체크박스로 감지됨
- **복잡한 그래픽**: 장식 요소나 비표준 폼 컨트롤
- **흐리거나 연한 색 요소**: 추출되지 않을 수 있음

form_structure.json에 없는 필드가 이미지에 있으면 해당 필드에 **시각적 분석**을 사용한다("하이브리드 방법" 참조).

### A.3: PDF 좌표로 fields.json 작성

각 필드의 입력 좌표를 추출된 구조에서 계산한다:

**텍스트 필드:**
- 입력 x0 = 레이블 x1 + 5 (레이블 이후 작은 간격)
- 입력 x1 = 다음 레이블의 x0, 또는 행 경계
- 입력 top = 레이블 top과 동일
- 입력 bottom = 아래 행 경계선, 또는 레이블 bottom + 행 높이

**체크박스:**
- form_structure.json의 체크박스 사각형 좌표 직접 사용
- entry_bounding_box = [checkbox.x0, checkbox.top, checkbox.x1, checkbox.bottom]

`pdf_width`와 `pdf_height`를 사용하여 fields.json 작성(PDF 좌표 신호):
```json
{
  "pages": [
    {"page_number": 1, "pdf_width": 612, "pdf_height": 792}
  ],
  "form_fields": [
    {
      "page_number": 1,
      "description": "성(last name) 입력 필드",
      "field_label": "Last Name",
      "label_bounding_box": [43, 63, 87, 73],
      "entry_bounding_box": [92, 63, 260, 79],
      "entry_text": {"text": "Smith", "font_size": 10}
    },
    {
      "page_number": 1,
      "description": "미국 시민권 Yes 체크박스",
      "field_label": "Yes",
      "label_bounding_box": [260, 200, 280, 210],
      "entry_bounding_box": [285, 197, 292, 205],
      "entry_text": {"text": "X"}
    }
  ]
}
```

**중요**: form_structure.json에서 가져온 좌표와 함께 `pdf_width`/`pdf_height`를 사용한다.

### A.4: 바운딩 박스 검증

채우기 전에 바운딩 박스 오류를 확인한다:
```bash
python ~/.config/opencode/skills/pdf/scripts/check_bounding_boxes.py fields.json
```

교차하는 바운딩 박스나 폰트 크기에 비해 너무 작은 입력 박스를 확인하고 수정한다.

---

## B 방법: 시각적 추정 (대체 방법)

PDF가 스캔/이미지 기반이고 구조 추출에서 사용 가능한 텍스트 레이블을 찾지 못한 경우(예: 모든 텍스트가 "(cid:X)" 패턴으로 표시) 사용한다.

### B.1: PDF를 이미지로 변환

```bash
python ~/.config/opencode/skills/pdf/scripts/convert_pdf_to_images.py <input.pdf> <images_dir/>
```

### B.2: 초기 필드 파악

각 페이지 이미지를 검토하여 양식 섹션과 필드 위치의 **대략적인 추정**을 파악한다:
- 양식 필드 레이블과 대략적인 위치
- 입력 영역 (텍스트 입력을 위한 선, 박스, 빈 공간)
- 체크박스와 대략적인 위치

각 필드에 대해 대략적인 픽셀 좌표를 기록한다(정확하지 않아도 됨).

### B.3: 줌 세밀 보정 (정확도를 위해 필수)

각 필드에 대해 추정 위치 주변을 잘라내어 좌표를 정밀하게 보정한다.

**ImageMagick으로 확대 자르기:**
```bash
magick <page_image> -crop <width>x<height>+<x>+<y> +repage <crop_output.png>
```

**예시:** "Name" 필드가 (100, 150) 부근으로 추정되는 경우:
```bash
magick images_dir/page_1.png -crop 300x80+50+120 +repage crops/name_field.png
```

(참고: `magick` 명령이 없으면 동일한 인자로 `convert`를 사용한다.)

**잘라낸 이미지를 검토**하여 정확한 좌표를 결정한다:
1. 입력 영역이 정확히 시작하는 픽셀 위치 파악
2. 입력 영역이 끝나는 위치 파악
3. 입력 선/박스의 상단과 하단 파악

**자르기 좌표를 전체 이미지 좌표로 변환:**
- full_x = crop_x + crop_offset_x
- full_y = crop_y + crop_offset_y

### B.4: 세밀 보정 좌표로 fields.json 작성

`image_width`와 `image_height`를 사용하여 fields.json 작성(이미지 좌표 신호):
```json
{
  "pages": [
    {"page_number": 1, "image_width": 1700, "image_height": 2200}
  ],
  "form_fields": [
    {
      "page_number": 1,
      "description": "성(last name) 입력 필드",
      "field_label": "Last Name",
      "label_bounding_box": [120, 175, 242, 198],
      "entry_bounding_box": [255, 175, 720, 218],
      "entry_text": {"text": "Smith", "font_size": 10}
    }
  ]
}
```

**중요**: `image_width`/`image_height`와 줌 분석에서 얻은 픽셀 좌표를 사용한다.

### B.5: 바운딩 박스 검증

```bash
python ~/.config/opencode/skills/pdf/scripts/check_bounding_boxes.py fields.json
```

---

## 하이브리드 방법: 구조 + 시각적

구조 추출이 대부분의 필드에서 작동하지만 일부 요소(원형 체크박스, 비표준 폼 컨트롤 등)를 놓친 경우 사용한다.

1. form_structure.json에서 감지된 필드에는 **A 방법** 사용
2. 누락된 필드 시각적 분석을 위해 **PDF를 이미지로 변환**
3. 누락된 필드에 **줌 세밀 보정** (B 방법에서) 사용
4. **좌표 결합**: 구조 추출 필드는 `pdf_width`/`pdf_height` 사용, 시각 추정 필드는 이미지 좌표를 PDF 좌표로 변환:
   - pdf_x = image_x * (pdf_width / image_width)
   - pdf_y = image_y * (pdf_height / image_height)
5. **단일 좌표 시스템 사용** — 모두 `pdf_width`/`pdf_height` PDF 좌표로 변환

---

## 2단계: 채우기 전 검증

**항상 채우기 전에 바운딩 박스를 검증한다:**
```bash
python ~/.config/opencode/skills/pdf/scripts/check_bounding_boxes.py fields.json
```

확인 사항:
- 교차하는 바운딩 박스 (텍스트 겹침 유발)
- 지정한 폰트 크기에 비해 너무 작은 입력 박스

## 3단계: 양식 채우기

채우기 스크립트가 좌표 시스템을 자동 감지하고 변환을 처리한다:
```bash
python ~/.config/opencode/skills/pdf/scripts/fill_pdf_form_with_annotations.py <input.pdf> fields.json <output.pdf>
```

## 4단계: 출력 확인

채워진 PDF를 이미지로 변환하여 텍스트 배치를 확인한다:
```bash
python ~/.config/opencode/skills/pdf/scripts/convert_pdf_to_images.py <output.pdf> <verify_images/>
```

텍스트 위치가 맞지 않는 경우:
- **A 방법**: `pdf_width`/`pdf_height`와 함께 form_structure.json의 PDF 좌표를 사용하는지 확인
- **B 방법**: 이미지 크기가 일치하고 좌표가 정확한 픽셀인지 확인
- **하이브리드**: 시각 추정 필드의 좌표 변환이 올바른지 확인
