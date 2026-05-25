# EDA Summary

## 분석 대상

본 EDA는 hERG inhibition 예측에 사용한 세 가지 학습 feature set을 중심으로 수행하였다.

| 데이터 | Train 규모 | Test 규모 | Feature 수 | 설명 |
|---|---:|---:|---:|---|
| ChemBERTa feature | 39,994행, 771컬럼 | 9,998행, 770컬럼 | 768개 | SMILES 기반 ChemBERTa 임베딩 |
| RDKit descriptor | 39,994행, 213컬럼 | 9,998행, 212컬럼 | 210개 | RDKit으로 계산한 분자 descriptor |
| RDKit fingerprint | 39,994행, 2,051컬럼 | 9,998행, 2,050컬럼 | 2,048개 | Morgan fingerprint binary bit |

Train 데이터에는 공통적으로 `id`, `smiles`, `hERG_inhibition` 컬럼이 포함되어 있으며, test 데이터에는 정답 변수인 `hERG_inhibition`이 포함되어 있지 않다.

## 주요 점검 항목

* 데이터 크기와 train/test feature schema 일치 여부 확인
* 결측치, 비수치 값, 무한대 값 등 비정상 값 존재 여부 확인
* 중복 `id`, 중복 `smiles` 존재 여부 확인
* target 변수(`hERG_inhibition`)의 분포 및 이상값 범위 확인
* feature별 특성 확인
  * ChemBERTa: dense embedding feature
  * RDKit descriptor: 연속형 분자 특성 feature
  * RDKit fingerprint: 0/1 binary 구조 feature
* 상수 컬럼 또는 정보량이 낮은 컬럼 존재 여부 확인
* fingerprint의 경우 active bit 분포와 train/test 간 sparse 정도 확인

## 핵심 발견사항

* 세 데이터 모두 train/test 간 feature schema가 일치하였다.
* 결측치, 비수치 값, 무한대 값은 발견되지 않았다.
* 중복 `id`와 중복 `smiles`는 발견되지 않았다.
* train 데이터의 target 범위가 넓게 분포하였다.
  * 최소값: -118.7569
  * 최대값: 126.4892
  * 음수 target: 11,694개
* RDKit descriptor 일부 변수에서 상수 컬럼이 존재하였다.
  * train: 7개
  * test: 9개
* RDKit fingerprint는 매우 sparse한 binary feature였다.
  * train 분자당 active bit 평균: 45.99개
  * test 분자당 active bit 평균: 45.97개
  * train/test의 sparse 정도는 거의 동일하였다.
* fingerprint에서 동일한 2048-bit vector를 갖는 서로 다른 분자들이 일부 존재하였다. 이는 fingerprint-only 모델이 구분하기 어려운 구조적 한계로 볼 수 있다.

## 결측치 및 이상값 처리

* 결측치가 없으므로 별도의 결측치 대체는 수행하지 않았다.
* 비수치 값이나 무한대 값이 없어 추가적인 값 보정은 필요하지 않았다.
* 모델 학습 시 `id`, `smiles`는 식별 정보이므로 feature에서 제외하였다.
* `hERG_inhibition`은 train 데이터에서 target 변수로 분리하였다.
* target에 음수와 큰 이상값이 존재하므로 예측값을 단순히 0 이상으로 clipping하지 않았다.
* RDKit descriptor의 상수 컬럼은 모델 학습 전 제거하거나, tree/MLP 모델 학습 과정에서 정보량이 낮은 변수로 처리될 수 있다.

## 모델링 시사점

* 단일 feature set만 사용하는 것보다 RDKit descriptor, RDKit fingerprint, ChemBERTa embedding을 함께 활용하는 앙상블 전략이 효과적이었다.
* 최종 제출에서는 각 feature 특성에 맞춰 서로 다른 모델을 구성하였다.
  * RDKit descriptor: MLP
  * RDKit descriptor + fingerprint: LightGBM
  * fingerprint: feature importance 기반 top 500 선택 후 LightGBM
  * ChemBERTa embedding: MLP
* RDKit descriptor 기반 MLP가 단일 모델 중 가장 좋은 성능을 보여, descriptor feature가 hERG inhibition 예측에 강한 정보를 제공하였다.
* fingerprint 단독 모델은 성능이 상대적으로 낮았지만, descriptor와 결합하거나 앙상블에 포함했을 때 보완적인 정보를 제공하였다.
* ChemBERTa embedding 단독 성능은 가장 강하지 않았지만, 다른 feature set과 다른 표현을 제공하므로 최종 앙상블 구성에 활용되었다.
* 최종 성능 향상은 단일 모델 튜닝보다 여러 base model의 OOF 예측값을 결합한 weighted ensemble에서 크게 발생하였다.
* 평가 지표가 MAE이므로 학습과 검증에서도 MAE 기반 성능 확인이 중요하며, target 범위가 넓어 안정적인 K-Fold OOF 검증이 필요하다.
