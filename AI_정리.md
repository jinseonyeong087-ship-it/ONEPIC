# ONEPIC AI 전체 구성 정리

작성일: 2026-04-29

## 1) 이 프로젝트에서 실제 사용 중인 AI

### 1-1. 객체 탐지: **YOLO (Ultralytics)**
- 라이브러리: `ultralytics`
- 코드 위치: `backend/app/services/ai_service.py`
- 모델 파일(실사용): `backend/app/services/model_files/best.pt`
- 역할:
  - 업로드된 이미지에서 상품 영역(박스) 탐지
  - 여러 박스 중 confidence가 가장 높은 1개를 선택해서 후속 처리
- 주요 추론 파라미터:
  - `imgsz=352`
  - `conf=0.20`
  - `iou=0.50`

### 1-2. 상품 분류: **MobileNetV3 (PyTorch / TorchVision)**
- 라이브러리: `torch`, `torchvision`
- 코드 위치: `backend/app/services/ai_service.py`
- 모델 파일: `backend/app/services/model_files/mobilenetv3_best.pt`
- 역할:
  - YOLO로 잘라낸 crop 이미지를 클래스(총 34개)로 분류
  - 분류 결과 `class_id`를 DB 매핑 테이블로 실제 상품(`product_id`)에 연결
- 특징:
  - 체크포인트에서 `small/large` 백본을 자동 추론해 로드
  - softmax 확률로 confidence 계산

### 1-3. OCR(문자 인식): **PaddleOCR**
- 라이브러리: `paddleocr`, `paddlepaddle`
- 코드 위치: `backend/app/services/ai_service.py`
- 역할:
  - 분류 confidence가 낮을 때(기준 `< 0.90`) 보조적으로 텍스트 추출
  - 텍스트에서 용량(ml/l/g/kg) 패턴 추출 (`extract_size`)
- 설정:
  - `lang="korean"`
  - `use_angle_cls=True`
  - GPU 가능 시 GPU 사용

---

## 2) AI 요청/응답 흐름

### API 엔드포인트
- 라우터: `backend/app/routers/ai.py`
- 엔드포인트: `POST /detect` (상위 prefix 포함 시 `/api/ai/detect` 형태로 사용될 가능성)

### 처리 순서
1. 클라이언트가 이미지 업로드
2. YOLO로 객체 탐지
3. 최고 confidence 박스 crop
4. MobileNetV3로 `class_id` 분류
5. confidence 낮으면 OCR 수행
6. `AIProductMapping.class_id -> product_id` DB 매핑
7. `Product` 조회 + (옵션) flavor attribute 조회
8. `RecognitionLog`에 confidence 기록
9. 최종 JSON 반환

### 반환 필드(핵심)
- `product_id`, `product_name`, `brand_name`
- `image_url`, `size`, `price`
- `confidence`, `ocr_text`

---

## 3) DB와 AI 연결 구조

- 매핑 테이블: `backend/app/models/ai_product_mapping.py`
  - AI 분류 결과 `class_id`를 서비스 상품 `product_id`로 매핑
- 로그 테이블: `backend/app/models/recognition_log.py`
  - 인식 confidence 등을 기록

즉, 이 프로젝트는 **“모델이 바로 상품명 확정”** 구조가 아니라,
**모델은 class_id를 내고, 실제 상품은 DB 매핑으로 확정**하는 구조입니다.

---

## 4) 의존성 기준으로 확인된 AI 스택

`backend/requirements.txt` 기준:
- `torch`, `torchvision`
- `ultralytics`
- `opencv-python`, `numpy`
- `paddlepaddle`, `paddleocr`

---

## 5) 루트/문서 상 AI 관련 파일 현황

- 루트 `best.pt`: 전역에 있으나, 서비스 코드상 우선 경로는 `backend/app/services/model_files/best.pt`
- `read.md`, `presentation.md`: AI 구조 설명 문서
- `ai_old/`(문서상 언급): 과거/예비 AI 자산 저장 용도

---

## 6) 한 줄 요약

이 프로젝트의 AI는 **YOLO(탐지) + MobileNetV3(분류) + PaddleOCR(보조 OCR)** 조합이며,
최종 상품 결정은 **DB 매핑(class_id→product_id)** 으로 안정적으로 연결하는 구조입니다.
