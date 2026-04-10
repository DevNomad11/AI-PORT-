# AI-PORT: AI 기반 공항 보안검색 가상 대기열 시스템

공항 CCTV 영상을 AI로 실시간 분석하여 보안검색 대기 인원을 파악하고, 대기시간을 예측하며, 가상 대기열("캐치테이블") 합류 타이밍을 추천하는 시스템입니다.

**인천공항 AI-PORT 아이디어 공모전** (세션1 - 여객 서비스) 출품작

---

## 실행 방법

### 1. 압축 해제

다운로드한 zip 파일을 압축 해제하면 아래와 같은 구조가 됩니다:

```
aiport/
├── mvp/                              ← 소스코드
└── 223.실내외 군중 특성 데이터/        ← EXCO 7 샘플 데이터 (포함됨)
    └── 01-1.정식개방데이터/
        ├── Training/
        │   ├── 01.원천데이터/          (EXCO007 이미지 482장)
        │   └── 02.라벨링데이터/        (JSON 라벨)
        └── Validation/
            ├── 01.원천데이터/          (EXCO007 이미지 65장)
            └── 02.라벨링데이터/        (JSON 라벨)
```

### 2. 모델 가중치 다운로드

본 저장소의 [Releases](../../releases) 페이지에서 `best_model.pth`를 다운로드한 뒤,
`aiport/` 아래에 `checkpoints_apgcc/` 폴더를 만들어 배치합니다:

```
aiport/
├── checkpoints_apgcc/
│   └── best_model.pth          ← 여기에 배치
├── mvp/
└── 223.실내외 군중 특성 데이터/
```

### 3. 의존성 설치

Python 3.8 이상, CUDA 지원 GPU 환경이 필요합니다.

```bash
cd mvp
pip install -r requirements.txt
```

### 4. 실행

```bash
cd mvp
python run_catchtable.py --split train --cameras 7
```

이 명령은 동봉된 EXCO 7 카메라 데이터(482장)를 순서대로 처리하며, 각 프레임마다 다음을 수행합니다:

1. **APGCC 모델**로 대기열 구역 내 머리 포인트를 검출하여 인원수를 파악
2. **트래커**로 프레임 간 인물 이동을 추적
3. **캐치테이블 예측기**로 대기시간, 추세, 합류 추천을 계산
4. 결과를 시각화한 이미지를 저장

실행 중 터미널에 아래와 같은 로그가 출력됩니다:

```
  Frame  1/482 (Indoor_EXCO007_001.jpg):
    Count:  45 | Wait: 15.0 min | Trend: stable (+0.00/frame)
    -> wait: 대기열이 감소 추세입니다. 잠시 후 합류하세요.
```

GPU가 없는 환경에서는 `--device cpu`를 추가하세요 (속도가 느려집니다):
```bash
python run_catchtable.py --split train --cameras 7 --device cpu
```

### 5. 출력 확인

실행이 완료되면 `mvp/output/exco007/` 폴더에 결과가 저장됩니다:

```
mvp/output/exco007/
├── frame_000_Indoor_EXCO007_001.jpg    ← 시각화된 프레임 (구역, 머리 포인트, HUD)
├── frame_001_Indoor_EXCO007_002.jpg
├── ...
└── catchtable_results.json             ← 전체 프레임 분석 결과 (JSON)
```

**시각화 프레임 예시 내용:**
- 초록색 폴리곤: 대기열 구역
- 빨간 점: 검출된 머리 포인트
- 상단 HUD: 현재 인원수, 대기시간, 추세, 합류 추천

**catchtable_results.json 주요 필드:**

| 필드 | 설명 |
|------|------|
| `current_count` | 대기열 내 인원수 |
| `wait_minutes` | 예상 대기시간 (분) |
| `trend` | 추세 (`increasing` / `decreasing` / `stable`) |
| `trend_slope` | 프레임당 인원 변화율 |
| `optimal_join.action` | 합류 추천 (`join_now` / `wait`) |
| `optimal_join.reason` | 추천 사유 |

---

## 시스템 구조

```
CCTV 프레임
    │
    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  APGCC 모델  │────▶│  구역 기반   │────▶│  캐치테이블  │
│  (머리 포인트 │     │  인원 카운팅  │     │  대기시간 &  │
│   검출)       │     │  & 트래킹    │     │  합류 추천   │
└──────────────┘     └──────────────┘     └──────────────┘
```

- **APGCC 모델**: CCTV 이미지에서 사람의 머리 위치를 포인트로 검출
- **구역 기반 카운팅**: 사전 정의된 대기열 폴리곤 내 인원만 집계
- **캐치테이블 예측**: 인원 추세를 분석하여 대기시간과 최적 합류 시점을 추천

## 데이터셋

**EXCO 실내 CCTV** (카메라 EXCO 7, 고정 앵글)
- 출처: [AI Hub - 실내외 군중 특성 데이터](https://aihub.or.kr/)
- 학습 이미지 482장 + 검증 이미지 65장 (zip에 포함)
- 프레임별 포인트 어노테이션 (머리 위치, JSON)

## 주요 파일

| 파일 | 설명 |
|------|------|
| `run_catchtable.py` | 메인 실행 스크립트 |
| `config.py` | 구역 폴리곤, 모델 설정, 임계값 |
| `pipeline/predictor.py` | QueueMonitor + QueuePredictor |
| `pipeline/tracker.py` | 프레임 간 머리 트래커 |
| `pipeline/visualizer.py` | 시각화 렌더링 |
| `models/apgcc_model.py` | APGCC 모델 래퍼 |
| `apgcc/` | APGCC 모델 구현 (인코더-디코더) |
| `REPORT.md` | 공모전 제안서 |
