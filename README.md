# ESS 배터리 수명 예측

초기 100사이클의 충·방전 데이터를 이용해 배터리의 최종 수명(`cycle_life`)을 예측하고, 새로운 Batch에서도 유지되는 열화 신호를 찾는 프로젝트입니다.

## 프로젝트 개요

- 데이터셋: MIT-Stanford Battery Dataset (Severson et al., Nature Energy 2019)
- 학습 데이터: Batch 1 (`2017-05-12`)
- 평가 데이터: Batch 2 (`2018-02-20`)
- 추가 평가 데이터: Batch 3 (`2018-04-12`)
- 태스크: **Regression - Cycle Life 예측**
- 주요 평가지표: MAPE
- 보조 평가지표: MAE, RMSE

분류 기준인 550사이클을 적용하면 Batch 1의 클래스가 불균형하므로, 실제 수명을 연속값으로 예측하는 회귀를 선택했습니다.

## 파일 구조

```text
├── data/                         # 원본 .mat 데이터, Git 제외
├── processed/                    # 전처리 결과, Git 제외
├── figures/                      # 최종 시각화
├── output/pdf/                   # 제출 PDF
├── 01_Day1_EDA.ipynb             # Batch별 EDA와 모델 전략
├── 30-ESSHealth-scratch.ipynb    # 제공된 기본 분석 노트북
├── .gitignore
└── README.md
```

## 환경 설정

```bash
git clone https://github.com/팀명/ess-battery-project.git
cd ess-battery-project

python3 -m venv venv
source venv/bin/activate
pip install numpy pandas matplotlib seaborn scipy mat73 h5py scikit-learn jupyterlab

jupyter lab
```

원본 데이터는 저장소에 포함하지 않습니다. 아래 파일을 `data/`에 준비해야 합니다.

```text
data/
├── 2017-05-12_batchdata_updated_struct_errorcorrect.mat
├── 2018-02-20_batchdata_updated_struct_errorcorrect.mat
├── 2018-04-03_varcharge_batchdata_updated_struct_errorcorrect.mat
└── 2018-04-12_batchdata_updated_struct_errorcorrect.mat
```

## EDA

### Cycle Life 분포

- Batch 1: 평균 844.7, 중앙값 858.5, 장수명(`>1,000`) 21.7%
- Batch 2: 평균 565.7, 중앙값 472.0, 단수명(`<500`) 59.6%
- Batch 3: 평균 1,059.7, 중앙값 1,005.5, 장수명 50.0%
- 핵심 발견: Batch별 수명 분포가 크게 달라 외부 Batch를 이용한 일반화 평가가 필요합니다.

### 열화 곡선 분석

- 셀별 방전용량 `QD`와 초기용량으로 정규화한 `SOH`를 비교했습니다.
- 초기 100사이클의 변화량·기울기와 Knee point 후보를 계산했습니다.
- 핵심 발견: 절대용량보다 SOH 변화량과 기울기가 셀·Batch 간 초기용량 차이를 줄이는 데 적합합니다.

### ΔQ(V) 곡선 분석

- 약 10사이클과 100사이클의 `Qdlin` 차이를 계산했습니다.
- 장수명·단수명 셀의 ΔQ 형태와 분산을 비교했습니다.
- 핵심 발견: `delta_q_log_variance`, `delta_q_std`는 초기 열화를 요약하는 주요 피처 후보입니다.

### 충전 속도와 수명의 관계

- 충전 프로토콜별 평균 수명과 표본 수를 비교했습니다.
- 충전시간, C-rate, 온도와 수명의 상관관계를 확인했습니다.
- 핵심 발견: 정책별 수명 차이가 관찰되지만 표본 수가 적으므로 인과관계로 단정할 수 없습니다.

### Batch 간 비교

- 전체 139개 셀 중 수명 Target이 있는 셀은 129개입니다.
- Batch 2의 8개 셀과 Batch 3의 2개 셀은 `cycle_life`가 없어 Target 기반 분석에서 제외했습니다.
- 세 Batch에서 상관 방향이 유지되는 피처를 우선 선택하고, 특정 Batch에서만 강한 피처는 제외 후보로 분류했습니다.

## Modeling

### 피처 엔지니어링 전략

초기 100사이클 안에서 계산할 수 있는 피처만 사용합니다.

- ΔQ(V): 로그 분산, 표준편차, 최솟값
- 용량 열화: QD·SOH 변화량과 기울기
- 내부저항: IR 평균, 표준편차, 기울기
- 온도: 평균온도, 온도 범위
- 충전 조건: 평균 충전시간, 1·2단계 C-rate, 전환 SOC

`cycle_life`, 전체 관측 사이클 수, 최종 EOL 시점처럼 예측 당시 알 수 없는 값은 입력에서 제외합니다.

### 모델 선택 및 근거

- 후보 모델: DummyRegressor, Ridge, ElasticNet, RandomForestRegressor, GradientBoostingRegressor
- 우선 모델: Ridge Regression
- 선택 이유: 셀 수가 적고 피처 간 다중공선성이 존재하므로 규제가 있는 단순 모델을 기준으로 사용합니다.
- 비교 모델: 트리 기반 모델로 비선형 관계와 피처 상호작용을 확인합니다.

## 성능 결과

| Model | Batch 1 CV MAPE | Batch 1 Valid MAPE | Batch 2 Test MAPE | Batch 3 Test MAPE |
|---|---:|---:|---:|---:|
| DummyRegressor | - | - | - | - |
| Ridge | - | - | - | - |
| ElasticNet | - | - | - | - |
| RandomForest | - | - | - | - |
| GradientBoosting | - | - | - | - |

모델링 완료 후 `-`를 실제 결과로 교체합니다.

## 오류 분석

- 가장 큰 예측 오차를 보인 셀: `[모델링 후 작성]`
- 공통 특징: `[충전 정책, 온도, 초기 열화 패턴 등]`
- 원인 가설: Batch 간 분포 차이, 소표본, 측정 누락 또는 비선형 열화
- 개선 방향: 피처 안정성 검증, Group 기반 교차검증, 추가 Batch 학습 데이터 확보

## ESS 도메인 해석

실제 BESS 운영에서는 다음 의사결정에 활용할 수 있습니다.

- 조기 수명 예측을 통한 셀 교체와 정비 시점 계획
- 단수명 위험 셀의 조기 선별
- 충전 정책별 열화 위험 비교
- 운영 온도와 충전속도 최적화

한계:

- 실험실 셀 데이터와 실제 ESS 운전환경의 차이
- Batch별 충전 정책과 수명 분포 차이
- 작은 표본 수와 일부 Target 누락
- 실제 배포를 위한 운전 데이터, 환경 정보, 안전성 검증 추가 필요

## 참고문헌

- Severson et al. (2019). Data-driven prediction of battery cycle life before capacity degradation. *Nature Energy*, 4, 383-391.

## 팀 구성

- `[팀원 1]`: EDA, 피처 엔지니어링, 모델 개발, Batch 2 성능 평가
- `[팀원 2]`: EDA, 피처 엔지니어링, 모델 개발, Batch 3 성능 평가
