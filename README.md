# Archived legacy repository

This repository is preserved for history. The maintained LG Aimers project is
[lg-aimers-pitch-control](https://github.com/tswaincae1221/lg-aimers-pitch-control).

---
# LG Aimers 피처·모델 자동 실험실

기존 V1 155개 피처와 V1.1 `asof` 추세 실험을 확장한 설정 기반 실험
프레임워크입니다. 팀 공통 비교 기준은 항상 **2019~2023 학습 → 2024 검증**으로
고정합니다.

한 번의 실행으로 다음을 자동 처리합니다.

- V1 전처리와 Trackman 집계를 한 번만 수행
- 설정 파일에 적힌 피처 묶음 × 모델 조합을 순차 실행
- Brier Score, Log Loss, AUC, BSS, calibration 오차, 시간 기록
- V1 LightGBM 대비 개선량과 개선률 계산
- 실행별 예측값, calibration, 피처 중요도, 오류 보존
- 전체 리더보드, 개선폭, 점수-시간 그래프와 Markdown 보고서 생성
- 완료 조합 자동 건너뛰기와 중단 후 이어서 실행

## 가장 쉬운 실행

1. 이 저장소 내용을 GitHub 비공개 저장소에 올립니다.
2. `notebooks/run_experiment_lab_colab.ipynb`를 Colab에 업로드합니다.
3. 노트북 첫 설정 셀에서 GitHub 저장소 이름만 맞춥니다.
4. `MODE="quick"`, `PRESET="starter"`로 위에서부터 실행합니다.
5. quick 실행이 통과하면 `MODE="full"`로 바꿔 정식 비교합니다.

노트북의 기본 데이터 폴더는 다음과 같습니다.

```text
/content/drive/MyDrive/aimers_data/
├─ train.csv
├─ test.csv
└─ trackman_history.csv
```

## 제공 피처 블록

| 블록 | 개수 | 내용 |
|---|---:|---|
| `asof_trend` | 13 | 1·3·5경기 추세, 투수-타자 차이, strike-ball 차이 |
| `situation` | 11 | base-out, count-base-out, LI·후반·접전 압박 상호작용 |
| `pitchmix` | 8 | 구종 구성 entropy, 주 구종 비율, 구종 간 사용률 차이 |
| `reliability` | 7 | 투수·타자·Trackman 표본 신뢰도와 정보원 개수 |

모든 추가 블록은 현재 행의 공식 사전 정보나 `season < 예측 시즌`으로 만들어진
V1 피처만 사용합니다. 현재 투구의 정답은 사용하지 않습니다.

## 제공 모델

| 모델 설정 | 기본 프리셋 | 특징 |
|---|---|---|
| Constant | starter | 학습 성공률 기준선 |
| LightGBM base | starter | 기존 V1과 동일한 기준 모델 |
| LightGBM regularized | extended | 더 강한 규제와 큰 leaf 최소 표본 |
| LightGBM wide | extended | 더 많은 상호작용 탐색 |
| Logistic Regression | extended | 빈도 인코딩 선형 기준선 |
| HistGradientBoosting | extended | scikit-learn 히스토그램 부스팅 |
| ExtraTrees | extended | 배깅 기반 비선형 비교군 |
| XGBoost | all | 선택 설치 모델 |
| CatBoost | all | 선택 설치, 범주형 직접 처리 |

`starter`는 피처 블록의 효과를 같은 LightGBM으로 비교합니다. `extended`는
모델과 제거 실험까지 넓히며, `all`은 설치 시간이 긴 XGBoost와 CatBoost도
포함합니다.

## 직접 실행

```bash
python -m pip install -r requirements.txt -r requirements-dev.txt
python -m pytest -q

python -m src.experiment_runner \
  --config config/experiments.json \
  --train data/train.csv \
  --test data/test.csv \
  --trackman data/trackman_history.csv \
  --mapping resources/pitcher_trackman_mapping.csv \
  --output-dir results/experiment_lab/quick \
  --mode quick \
  --preset starter \
  --validation-season 2024 \
  --n-jobs 2
```

정식 비교는 출력 폴더와 모드만 바꿉니다.

```bash
python -m src.experiment_runner \
  --config config/experiments.json \
  --train data/train.csv \
  --test data/test.csv \
  --trackman data/trackman_history.csv \
  --mapping resources/pitcher_trackman_mapping.csv \
  --output-dir results/experiment_lab/full \
  --mode full \
  --preset starter \
  --validation-season 2024 \
  --n-jobs 4
```

특정 조합만 실행하려면 `--only`를 사용합니다.

```bash
python -m src.experiment_runner ... \
  --only v1__lgbm_base v1_asof__lgbm_base
```

## 실험 추가

`config/experiments.json`은 세 부분으로 나뉩니다.

1. `feature_sets`: 사용할 블록과 제거 패턴
2. `models`: 모델 종류와 하이퍼파라미터
3. `experiments`: 피처 묶음과 모델의 실제 조합

예를 들어 새 LightGBM 설정을 비교하려면 `models`에 설정을 추가한 뒤
`experiments`에 다음 형태의 항목을 추가합니다.

```json
{
  "name": "v1_asof__lgbm_custom",
  "feature_set": "v1_asof",
  "model": "lgbm_custom",
  "presets": ["extended"]
}
```

새 row-wise 피처는 `src/experiment_features.py`에 builder를 추가하고
`FEATURE_BLOCKS`에 등록합니다. 모델 실행 코드를 고치지 않고 설정에서 조합할 수
있습니다.

## 결과 구조

```text
results/experiment_lab/full/
├─ experiment_history.csv          # 성공·실패를 포함한 누적 기록
├─ leaderboard.csv                 # 동일 데이터·모드의 최신 결과
├─ experiment_report.md            # 자동 해석 보고서
├─ feature_catalog.csv             # 추가 피처 목록
├─ leaderboard_brier.png
├─ improvement_vs_baseline.png
├─ score_vs_time.png
├─ cache/                          # Trackman 집계 재사용
└─ runs/
   └─ 실행시각__실험명__설정해시/
      ├─ metrics.json
      ├─ resolved_config.json
      ├─ feature_report.json
      ├─ validation_predictions.csv.gz
      ├─ calibration_bins.csv
      ├─ calibration_curve.png
      ├─ prediction_distribution.png
      ├─ feature_importance.csv
      ├─ feature_importance_top30.png
      └─ error.txt                 # 실패한 실험에만 생성
```

판정은 `brier_delta_vs_baseline`을 보면 됩니다.

- 음수: V1 LightGBM보다 개선
- 0: 동일
- 양수: 악화

quick 모드는 시즌당 일부 행만 쓰므로 점수 비교용이 아닙니다. 최종 채택은 반드시
full 모드 결과로 결정합니다.

## 대회 데이터 보호

대회 원본 데이터, 전처리 결과, 실행 결과와 학습 모델은 저장소에 포함하지 않습니다.
대회 규정을 재확인하기 전까지 저장소는 비공개로 유지하는 것을 권장합니다.
