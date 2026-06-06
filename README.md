# 2026 MixUP 3rd AI Datathon Track 2 대회 정리
2026년 5월 참여한 '2026 MixUP 3rd AI Datathon Track 2' 대회의 코드 파일 정리 및 기록 문서입니다.


## 📋 대회 개요
* 대회명: 2026 MixUP 3rd AI Datathon Track 2
* 주최/주관: Prometheus X BITAmin X TOBIG's / 한국생명공학연구원
* 후원/협찬: UPSTAGE, 유가네닭갈비, MONSTER ENERGY, LOVABLE
* 참여기간: 2026년 5월 22일 (금) 21:00 ~ 5월 23일 (토) 09:00
* 대회 링크: [DACON 공식 페이지](https://dacon.io/competitions/official/236720/overview/description)
* 문제 유형: 분자 구조 기반 hERG inhibition at 1µM 회귀 예측 AI 모델 개발


## 🎯 문제 정의 및 데이터 설명
### 문제 배경
분자 구조 기반 hERG inhibition at 1µM 회귀 예측 AI 모델 개발

### 데이터 구성

#### 학습 데이터
* 파일명: train_chemberta_feature.csv / train_rdkit_descriptor.csv / train_rdkit_fingerprint.csv
* 규모: 공통 39,994행, 771개 컬럼 / 213개 컬럼 / 2,051개 컬럼

#### 주요 변수 설명
| 변수 그룹 | 변수명 | 설명 |
|---|---|---|
| 식별자 | id | 각 분자 샘플을 구분하는 고유 ID |
| 분자 구조 | smiles | 분자 구조를 문자열로 표현한 SMILES 정보 |
| 타겟 변수 | hERG_inhibition | hERG 저해 활성값, 학습 모델이 예측해야 하는 정답 변수 |
| ChemBERTa 임베딩 | chemberta_0 ~ chemberta_767 | SMILES를 기반으로 ChemBERTa 모델이 추출한 768차원 분자 표현 벡터 |
| RDKit Descriptor | MaxAbsEStateIndex, qed, MolWt, MolLogP, TPSA, fr_* 등 | RDKit으로 계산한 분자의 물리화학적 성질, 구조적 특징, 작용기 관련 수치 변수 |
| Morgan Fingerprint | morgan_r2_bit_0 ~ morgan_r2_bit_2047 | Morgan fingerprint 반지름 2 기준으로 생성한 2048개 이진 구조 비트 변수 |

#### 테스트 데이터
* 파일명: test_chemberta_feature.csv / test_rdkit_descriptor.csv / test_rdkit_fingerprint.csv
* 규모: 공통 9,998행, 770개 컬럼 / 212개 컬럼 / 2,050개 컬럼

#### 평가 지표
* MAE, Mean Absolute Error
* MAE는 실제값과 예측값의 절대 오차 평균을 의미하며, 값이 낮을수록 예측 성능이 우수합니다.
* MAE = (1 / n) * Σ |y_i - ŷ_i|
* y_i : 실제 hERG inhibition 값
* ŷ_i : 모델이 예측한 hERG inhibition 값
* n : 평가 샘플 수


## 🔍 데이터 분석 및 전처리
### 탐색적 데이터 분석 (EDA)

#### 주요 점검 항목

* 데이터 크기와 train/test feature schema 일치 여부 확인
* 결측치, 비수치 값, 무한대 값 등 비정상 값 존재 여부 확인
* 중복 id, 중복 smiles 존재 여부 확인
* target 변수(hERG_inhibition)의 분포 및 이상값 범위 확인
* feature별 특성 확인
  * ChemBERTa: dense embedding feature
  * RDKit descriptor: 연속형 분자 특성 feature
  * RDKit fingerprint: 0/1 binary 구조 feature
* 상수 컬럼 또는 정보량이 낮은 컬럼 존재 여부 확인

#### 핵심 발견사항

* 세 데이터 모두 train/test 간 feature schema가 일치함
* 결측치, 비수치 값, 무한대 값은 발견되지 않음
* 중복 id와 중복 smiles는 발견되지 않음
* train 데이터의 target 범위가 넓음
  * 최소값: -118.7569
  * 최대값: 126.4892
  * 음수 target: 11,694개
* RDKit descriptor 일부 변수에서 상수 컬럼이 존재함
  * train: 7개
  * test: 9개
* fingerprint는 매우 sparse한 binary feature이며, ChemBERTa와 descriptor는 수치형 feature로 구성됨

#### 결측치 및 이상값 처리

* 결측치가 없으므로 별도의 결측치 대체는 수행하지 않음
* 비수치 값이나 무한대 값이 없어 추가적인 값 보정은 필요하지 않음
* 모델 학습 시 id, smiles는 식별 정보이므로 feature에서 제외
* hERG_inhibition은 train에서 target 변수로 분리
* target에 음수와 큰 이상값이 존재하므로 예측값을 단순히 0 이상으로 clipping하지 않음
* RDKit descriptor의 상수 컬럼은 모델 학습 전 제거 가능

#### 모델링 시사점

* 단일 feature set만 사용하는 것보다 RDKit descriptor, RDKit fingerprint, ChemBERTa embedding을 함께 활용하는 앙상블 전략이 효과적임
* 최종 제출에서는 각 feature 특성에 맞춰 서로 다른 모델을 구성함
  * RDKit descriptor: MLP
  * RDKit descriptor + fingerprint: LightGBM
  * fingerprint: feature importance 기반 top 500 선택 후 LightGBM
  * ChemBERTa embedding: MLP
* RDKit descriptor 기반 MLP가 단일 모델 중 가장 좋은 성능을 보여, descriptor feature가 hERG inhibition 예측에 강한 정보를 제공함
* fingerprint 단독 모델은 성능이 상대적으로 낮았지만, descriptor와 결합하거나 앙상블에 포함했을 때 보완적인 정보를 제공함
* ChemBERTa embedding 단독 성능은 가장 강하지 않았지만, 다른 모델과 다른 표현을 제공하므로 최종 앙상블 구성에 활용됨
* 최종 성능 향상은 단일 모델 튜닝보다 여러 base model의 OOF 예측값을 결합한 weighted ensemble에서 크게 발생함
* 따라서 최종 모델링 전략은 하나의 최강 모델을 선택하는 방식보다, 서로 다른 feature와 모델 구조의 예측을 조합하는 ensemble 방식이 적합함
* 평가 지표가 MAE이므로 학습과 검증에서도 MAE 기반 성능 확인이 중요하며, target 범위가 넓어 안정적인 K-Fold OOF 검증이 필요함


## 🤖 모델링 및 실험
### 모델링 파이프라인

hERG inhibition 예측을 위해 하나의 모델에 의존하기보다, 서로 다른 분자 표현을 활용한 여러 모델을 학습한 뒤 최종 앙상블하는 방식으로 접근하였다.

* 데이터 전처리: `id`, `smiles`, `hERG_inhibition`을 제외하고 수치형 feature를 학습 변수로 사용
* 피처 구성: RDKit descriptor, RDKit fingerprint, ChemBERTa embedding을 각각 또는 조합하여 사용
* 기본 모델 실험: MLP와 LightGBM 기반 모델을 feature set별로 학습
* 피처 선택: fingerprint는 LightGBM feature importance를 기준으로 상위 500개 bit를 선택
* 모델 검증: 5-Fold KFold로 OOF 예측을 생성하고 MAE 기준으로 성능 평가
* 앙상블 구축: 각 base model의 OOF 예측값을 활용하여 최적 가중치 탐색
* 최종 예측: 최적 가중 앙상블 결과를 최종 submission으로 생성

### 모델 실험 결과

#### 실험 설정

* 평가 지표: MAE
* 교차 검증 전략: 5-Fold KFold
* Random Seed: 42
* 사용 feature:
  * RDKit descriptor 210개
  * Morgan fingerprint 2,048개
  * ChemBERTa embedding 768개
  * MACCS fingerprint 166개 추가 활용

#### 모델별 결과

| 모델 | 사용 변수 | OOF MAE |
|---|---|---:|
| Descriptor MLP | RDKit descriptor | 8.902794 |
| Desc + FP + MACCS MLP | descriptor + fingerprint + MACCS | 9.192788 |
| Desc + FP LightGBM | descriptor + fingerprint | 9.081495 |
| Fingerprint LightGBM FS500 | fingerprint 상위 500개 | 9.302554 |
| ChemBERTa MLP | ChemBERTa embedding | 9.432980 |
| 최종 앙상블 | 위 5개 모델 예측값 결합 | 8.533049 |

### 최종 모델 선정

최종 모델은 단일 모델이 아니라, 5개 base model의 예측값을 결합한 weighted ensemble로 선정하였다.

* 최종 전략: Top5 Multi-start Nelder-Mead weighted ensemble
* 최종 OOF MAE: 8.533049
* Public LB: 8.8629921078
* 최종 제출 파일: submission_final_top5.csv

### 모델 선정 근거

여러 단일 모델 중에서는 RDKit descriptor 기반 MLP가 가장 좋은 성능을 보였지만, feature 표현 방식이 다른 모델들을 함께 사용했을 때 OOF MAE가 더 크게 개선되었다.

따라서 단일 최고 성능 모델을 선택하기보다, descriptor, fingerprint, ChemBERTa가 제공하는 서로 다른 정보를 조합하는 앙상블 전략을 최종 모델로 선택하였다. 이 방식은 개별 모델의 한계를 보완하고, 예측 안정성과 최종 성능을 높이는 데 효과적이었다.


## 📊 결과 및 인사이트

### 최종 성능

최종 모델은 단일 모델이 아닌, 5개 base model의 예측값을 결합한 weighted ensemble 방식으로 구성하였다.

* 최종 전략: Top5 Multi-start Nelder-Mead weighted ensemble
* 최종 OOF MAE: 8.533049
* Public LB: 8.8629921078
* 최종 제출 파일: `submission_final_top5.csv`

### 핵심 인사이트

#### 모델별 성능 분석

* RDKit descriptor 기반 MLP가 단일 모델 중 가장 우수한 성능을 보임
* Descriptor feature가 hERG inhibition 예측에 가장 강한 정보를 제공함
* Fingerprint 단독 모델은 상대적으로 성능이 낮았지만, descriptor와 결합하거나 앙상블에 포함했을 때 보완적인 역할을 수행함
* ChemBERTa embedding은 단독 성능은 낮았지만, 다른 feature set과 다른 표현 정보를 제공하여 앙상블 다양성 확보에 기여함

#### 피처 중요도 및 선택

* Morgan fingerprint 2,048개 bit 전체를 그대로 사용하는 대신, LightGBM feature importance를 기준으로 상위 500개 bit를 선택함
* 이를 통해 fingerprint feature의 차원을 줄이고, 예측에 상대적으로 중요한 구조 정보를 중심으로 모델을 학습함
* RDKit descriptor, fingerprint, ChemBERTa는 서로 다른 방식으로 분자를 표현하므로 조합 시 성능 개선 효과가 나타남

#### 주요 시사점

* hERG inhibition 예측에서는 단일 feature set보다 다양한 분자 표현을 함께 활용하는 것이 효과적임
* Descriptor 기반 모델이 가장 강한 baseline 역할을 수행하며, fingerprint와 ChemBERTa는 보완적인 정보를 제공함
* 최종 성능 향상은 개별 모델 튜닝보다 OOF 기반 앙상블에서 크게 발생함
* target 범위가 넓고 음수 값도 존재하므로, MAE 기반 검증과 안정적인 K-Fold OOF 평가가 중요함
* 최종적으로는 “가장 좋은 단일 모델”보다 “서로 다른 예측 패턴을 가진 모델 조합”이 더 높은 성능을 달성함


## 💡 프로젝트 회고

### 느낀 점/후기

hERG inhibition 예측은 단순히 하나의 모델 성능을 높이는 것보다, 분자를 어떤 방식으로 표현하느냐가 중요한 문제였다. RDKit descriptor, Morgan fingerprint, ChemBERTa embedding은 각각 다른 관점의 정보를 담고 있었고, 단일 모델만으로는 성능 개선에 한계가 있었다.

실험 결과 RDKit descriptor 기반 MLP가 단일 모델 중 가장 좋은 성능을 보였지만, fingerprint와 ChemBERTa 기반 모델을 함께 앙상블했을 때 최종 성능이 더 개선되었다. 이를 통해 서로 다른 feature representation을 조합하는 것이 hERG inhibition 예측에서 효과적임을 확인할 수 있었다.

최종적으로는 5개 base model의 OOF 예측값을 기반으로 Multi-start Nelder-Mead 방식의 weighted ensemble을 구성하였고, Public LB 8.8629921078을 달성하였다. 단일 모델 튜닝보다 다양한 모델의 예측 패턴을 안정적으로 결합하는 전략이 핵심이었다.

### 연사님 피드백
    
eda 더 하고, 해당 도메인에서의 중요한 피쳐들을 사용했으며 좋았을 것
GNN을 사용하지 않은 것은 아쉬움
음수값을 단순히 이상치로 처리하는 것은 위험, 특히 생물학에서는 더 주의할것
    
AI가 도메인지식까지 자동으로 알려주진 않는다. 그에 대해서는 더욱 공부하고 그를 AI에게 제시해야함
(코드는 AI가 짜지만, 도메인 지식과 아키텍쳐는 사람이 제시해야함)

### 핵심 성공 요인

* RDKit descriptor, fingerprint, ChemBERTa embedding을 모두 활용한 다양한 분자 표현 구성
* 5-Fold OOF 예측 기반의 안정적인 모델 검증
* Fingerprint feature importance 기반 top 500 feature selection 적용
* 단일 모델 성능에 의존하지 않고, 서로 다른 모델의 예측을 결합한 weighted ensemble 전략
* MAE 기준으로 모델을 비교하고, 최종 예측까지 동일한 평가 관점 유지


## 🛠️ 사용 도구 및 기술 스택

### 개발 환경

* 언어: Python
* 실행 환경: Google Colab(4T GPU)
* 협업 및 코드 관리: Notion

### 주요 라이브러리

* 데이터 처리: Pandas, NumPy
* 분자 특성 처리: RDKit
* 모델링:
  * PyTorch
  * LightGBM
  * Scikit-learn
* 교차 검증 및 평가:
  * KFold
  * Mean Absolute Error(MAE)
* 하이퍼파라미터 및 앙상블 최적화:
  * Scipy Optimize
  * Multi-start Nelder-Mead
* 시각화 및 분석:
  * Matplotlib
  * Seaborn
* 결과 저장 및 제출:
  * CSV 기반 submission 생성
