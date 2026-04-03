# AI-PORT: AI 기반 공항 보안검색 가상 대기열 시스템

공항 CCTV 영상을 AI로 실시간 분석하여 보안검색 대기 인원을 파악하고, 대기시간을 예측하며, 가상 대기열("캐치테이블")을 운영하여 승객이 물리적 줄 없이 대기할 수 있게 하는 시스템입니다.

**인천공항 AI-PORT 아이디어 공모전** (세션1 - 여객 서비스) 출품작입니다.

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
                                                │
                              ┌─────────────────┼─────────────────┐
                              ▼                 ▼                 ▼
                        디스플레이 페이지   키오스크 페이지     REST API
                       (실시간 현황판)    (대기열 등록)     (JSON 엔드포인트)
```

## 데이터셋

**EXCO 실내 CCTV** 데이터 (카메라 EXCO 7-12, 동일 고정 앵글):
- 학습 이미지 약 2,900장 + 검증 이미지 약 380장
- 프레임별 포인트 어노테이션 (머리 위치)
- 출처: [AI Hub - 실내외 군중 특성 데이터](https://aihub.or.kr/)

데이터셋 배치 경로:
```
aiport/
├── 223.실내외 군중 특성 데이터/
│   └── 01-1.정식개방데이터/
│       ├── Training/
│       │   ├── 01.원천데이터/    (이미지)
│       │   └── 02.라벨링데이터/  (라벨)
│       └── Validation/
│           ├── 01.원천데이터/
│           └── 02.라벨링데이터/
└── mvp/                          (본 디렉토리)
```

## 요구사항

- Python 3.8 이상
- CUDA 지원 GPU (권장)

## 설치

```bash
cd mvp
pip install -r requirements.txt
```

모든 의존성(학습, 추론, 웹 데모 포함)이 `requirements.txt`에 포함되어 있습니다.

## 모델 체크포인트

학습된 모델 가중치 파일(`.pth`)은 용량 문제로 본 패키지에 포함되어 있지 않습니다.
GitHub Releases에서 다운로드하거나, 직접 학습하여 생성할 수 있습니다.

**다운로드:**
본 저장소의 [Releases](../../releases) 페이지에서 `best_model.pth`를 다운로드한 뒤 아래 경로에 배치하세요:

```
aiport/
├── checkpoints_apgcc/
│   └── best_model.pth      # APGCC 모델 (추론/웹 데모에 필요)
└── mvp/
```

**직접 학습:**
체크포인트 없이 학습부터 시작하려면 아래 [사용법 > 1. 모델 학습](#1-모델-학습) 섹션을 참고하세요.

## 프로젝트 구조

```
mvp/
├── config.py                # 구역 폴리곤, 모델 설정, 임계값 정의
├── requirements.txt
│
├── apgcc/                   # APGCC 모델 구현 (인코더-디코더)
│   ├── models/              #   APGCC, Encoder, Decoder, 백본
│   ├── datasets/            #   데이터셋 & 라벨 전처리
│   ├── configs/             #   YAML 학습 설정 파일
│   └── engine.py            #   학습/평가 엔진
│
├── models/                  # 모델 래퍼
│   ├── apgcc_model.py       #   APGCC 추론용 래퍼
│   └── density.py           #   CSRNet 밀도 추정 모델
│
├── data/                    # 데이터 로더
│   ├── dataset_apgcc.py     #   APGCC 데이터로더 (포인트 어노테이션)
│   └── dataset_density.py   #   밀도 맵 데이터로더
│
├── pipeline/                # 핵심 추론 파이프라인
│   ├── predictor.py         #   QueueMonitor + QueuePredictor
│   ├── tracker.py           #   프레임 간 머리 트래커
│   ├── monitor.py           #   구역별 카운팅 & 대기시간 추정
│   └── visualizer.py        #   draw_results() 시각화 렌더링
│
├── web/                     # FastAPI 웹 애플리케이션
│   ├── app.py               #   메인 앱 + 시작 시 모델 로딩
│   ├── inference_loop.py    #   백그라운드 추론 루프 (이미지 재생)
│   ├── queue_manager.py     #   가상 대기열 (캐치테이블) 로직
│   ├── routers/
│   │   ├── api.py           #   REST 엔드포인트 (/api/status, /api/join 등)
│   │   └── ws.py            #   WebSocket 브로드캐스트
│   ├── templates/           #   HTML 페이지 (디스플레이, 키오스크)
│   └── static/              #   CSS, JS
│
├── train.py                 # CSRNet 밀도 모델 학습
├── train_apgcc.py           # APGCC 포인트 검출 모델 학습
├── train_apgcc_colab.ipynb  # Google Colab 학습 노트북
├── run_inference.py         # CLI 추론 (단일 이미지 / 배치)
├── run_catchtable.py        # CLI 캐치테이블 데모 (카메라 시퀀스 처리)
│
├── flow.png                 # 입출구 방향 참고 다이어그램
├── zone.png                 # 대기열 구역 폴리곤 참고 이미지
├── REPORT.md                # 공모전 제안서
└── DEVELOPMENT.md           # 개발 로그
```

## 사용법

### 1. 모델 학습

**APGCC (포인트 검출, 권장):**
```bash
python train_apgcc.py --epochs 500 --batch_size 8
```

**CSRNet (밀도 추정):**
```bash
python train.py --model csrnet --epochs 50 --batch_size 8
```

체크포인트 저장 위치: `../checkpoints_apgcc/` (APGCC) 또는 `./checkpoints/` (CSRNet)

Google Colab 노트북도 제공됩니다: `train_apgcc_colab.ipynb`

### 2. 추론 실행 (CLI)

**단일 이미지:**
```bash
python run_inference.py --input /path/to/image.jpg --output result.jpg --model_type apgcc
```

**배치 (디렉토리):**
```bash
python run_inference.py --input /path/to/images/ --output /path/to/output/ --model_type apgcc
```

`--save_json` 옵션으로 결과를 JSON으로 저장할 수 있습니다. GPU가 없는 경우 `--device cpu`를 사용하세요.

### 3. 캐치테이블 데모 실행 (CLI)

EXCO 카메라 시퀀스를 처리하여 대기시간 예측 및 합류 추천을 수행합니다:

```bash
# 전체 학습 카메라 (EXCO 7-12)
python run_catchtable.py --split train

# 특정 카메라만
python run_catchtable.py --split train --cameras 7 8

# 임의 이미지 디렉토리
python run_catchtable.py --input /path/to/frames/ --name my_camera
```

출력: 어노테이션된 프레임 + `catchtable_results.json` → `./output/<camera_name>/`

### 4. 웹 데모

FastAPI 웹 서버를 실행합니다:
```bash
cd mvp
python -m web --image-dir /path/to/image/folder --port 8000
```

브라우저에서 접속:
- **디스플레이 페이지** (실시간 현황판): `http://localhost:8000/display`
- **키오스크 페이지** (승객 대기열 등록): `http://localhost:8000/kiosk`
- **API 문서** (Swagger): `http://localhost:8000/docs`

웹 데모는 지정된 디렉토리의 이미지를 순차 재생하여 실시간 CCTV 피드를 시뮬레이션합니다.

## 주요 출력

| 출력 항목 | 설명 |
|-----------|------|
| 인원수 (Head count) | 대기열 구역 내 검출된 인원 수 |
| 대기시간 (Wait time) | 인원수와 서비스 처리율 기반 예상 대기 분 |
| 추세 (Trend) | 대기열 증가 / 감소 / 안정 추세 |
| 합류 추천 (Join advice) | "지금 합류" / "대기" 추천 및 사유 |
| 시각화 (Visualization) | 구역 오버레이, 머리 포인트, HUD가 표시된 프레임 |

## 설정

모든 주요 파라미터는 [config.py](config.py)에서 관리합니다:

- `QUEUE_ZONES` — 대기열 구역 폴리곤 좌표 (1920x1080 기준)
- `ZONE_EDGES` — 입구/출구 경계선 정의
- `APGCC_CONFIG` — 모델 하이퍼파라미터
- `CATCHTABLE_CONFIG` — 서비스 처리율, 추세 분석 윈도우, 알림 임계값
