# Same Case, Different Reasons

같은 사안, 다른 이유: LLM 기반 설명 생성의 내용 변동성 분해

동일한 사례에 대해 LLM이 XAI 근거를 바탕으로 생성한 설명이 반복 생성 시에도
동일한 핵심 근거를 제시하는지 측정하고, 변동이 파이프라인의 어느 단계에서
발생하는지 분해한다.

---

## 폴더 구조

```
project/
├── data/                  원본·중간·분석용 데이터 (버전관리 제외)
├── notebooks/             분석 노트북 11개
├── prompts/               프롬프트 버전 관리
├── logs/                  실행 기록, 하이퍼파라미터, 실패 로그
├── results/
│   ├── figures/           PNG + PDF, 600 dpi
│   └── tables/            CSV
├── CODING_GUIDE.md        사람 부호화 지침서
├── requirements.txt
├── .env.example
└── README.md
```

---

## 환경 구성

```bash
conda create -n xai-stability python=3.11 -y
conda activate xai-stability

pip install -r requirements.txt
conda install -c conda-forge llvm-openmp -y      # macOS 필수

python -m ipykernel install --user --name xai-stability --display-name "XAI Stability"
```

주피터에서 커널을 **XAI Stability**로 지정하십시오.

### 버전 제약 — 임의로 풀지 마십시오

| 패키지 | 제약 | 이유 |
|---|---|---|
| `numpy` | `<2.4` | shap이 의존하는 numba가 2.4를 지원하지 않음 |
| `lightgbm` | `>=4.5` | 이하 버전은 scikit-learn 1.6에서 제거된 인자를 호출 |

### macOS

XGBoost와 LightGBM은 OpenMP 런타임을 요구합니다. conda-forge에서의 패키지명은
`libomp`가 아니라 **`llvm-openmp`** 입니다.

```bash
conda install -c conda-forge llvm-openmp
```

### API 키

```bash
cp .env.example .env
```

`.env`를 편집해 키를 넣으십시오. 노트북이 `python-dotenv`로 읽습니다. 키를
노트북 안에 직접 쓰지 마십시오 — 코드를 공개할 예정이기 때문입니다.

---

## 데이터

`data/`에 공개 데이터 두 개를 넣습니다.

| 파일명 | 출처 | 행 수 |
|---|---|---|
| `credit_risk_dataset.csv` | kaggle.com/datasets/laotse/credit-risk-dataset | 32,581 |
| `telco_churn.csv` | kaggle.com/datasets/blastchar/telco-customer-churn | 7,043 |

Telco 쪽은 원본 파일명이 `WA_Fn-UseC_-Telco-Customer-Churn.csv`이므로 변경이
필요합니다. 파일명이 다르면 노트북 01의 `DATASETS` 설정만 바꾸면 됩니다.

**주의**: Kaggle에 `credit risk dataset`이라는 이름이 여럿 있습니다. `laotse`
버전이어야 합니다. 타깃 `loan_status`가 `0`/`1`이면 맞고, `Fully Paid` 같은
텍스트면 다른 데이터셋입니다.

---

## 실행 순서

| # | 노트북 | 하는 일 | 주요 산출물 |
|---|---|---|---|
| 01 | `data_prep` | 전처리, train/test 분할 | `feature_registry.json` |
| 02 | `model_training` | 3개 모형 학습, AUC 동등성 확보 | `model_*.joblib` |
| 03 | `attribution` | 사례 층화표집, SHAP/LIME 산출 | `evidence_blocks.json` |
| 04 | `claim_parser_dev` | 파일럿 생성, 추출기 개발·안정성 측정 | `manual_coding_sheet.csv` |
| 05 | `parser_validation` | 사람 부호화 대조, 게이트 판정 | `validation_summary.json` |
| 06 | `power_simulation` | 표본 규모 사전 검토 | `design_recommendation.json` |
| 07 | `generation` | 본 실험, 약 33,600건 생성 | `generation_corpus.parquet` |
| 08 | `content_distance` | 주장 추출, 쌍별 거리 산출 | `content_distances.parquet` |
| 09 | `delta_analysis` | **주 분석** — 기준선 대비 추가 변동 | `delta_manifest.json` |
| 10 | `mixed_effects` | 강건성 검증 | `robustness_manifest.json` |
| 11 | `quality_stability` | RQ5 — 품질 지표의 사각지대 | `quality_scores.parquet` |

### 사람이 개입하는 지점

04 → 05 사이에 **사람 부호화**가 들어갑니다. 코드 작성이 아니라 설명 문장을
읽고 표에 옮기는 작업입니다. `CODING_GUIDE.md`를 코더에게 주십시오.

```
1. 04가 data/manual_coding_sheet.csv 생성
2. 코더 2인이 coder1_*, coder2_* 열을 독립적으로 채움
3. 05 실행 → data/adjudication.csv 생성
4. 불일치 항목 조정
5. 05 재실행 → 게이트 판정
```

**05의 게이트를 통과하지 못하면 07을 실행하지 마십시오.**

코더는 도메인 전문성이 필요 없습니다. 영어 독해와 지침 준수 능력이면 충분하고,
30분 교육으로 시작할 수 있습니다. 자동 추출 결과는 `coding_auto_reference.csv`에
분리 보관되므로 코더에게 노출되지 않습니다.

---

## 주장 추출기

**LLM 단독**입니다. 규칙 기반 파서를 병행하다 폐기했습니다.

`Contract_Two_year` 같은 원핫 변수명이 설명에는 "a two-year contract"로
나타나는데, 이를 규칙으로 잡으려면 데이터셋별 동의어 사전을 파일럿에 맞춰
계속 손봐야 했습니다. 그 손질은 **나중에 검증할 데이터에 도구를 미리 맞추는**
셈이었고, 그렇게 해도 재현율이 제공 변수의 3분의 1 수준에 머물렀습니다. LLM
추출기는 그런 튜닝 없이 거의 전부를 찾아냅니다.

대가는 **완전한 결정론적 재현성을 잃는 것**입니다. 온도 0에서도 LLM은 동일
출력을 보장하지 않습니다. 재현성을 주제로 하는 연구이므로 이를 두 가지로
보완합니다.

1. **추출 캐시 고정** — `data/claims_cache/`. 한 번 추출하면 재실행해도 동일한
   결과가 쓰입니다. 이 캐시를 공개하면 제3자가 같은 주장 데이터로 분석을
   재현할 수 있습니다.
2. **추출기 자체 재현성 측정** — 04번이 50건을 3회씩 재추출해 변수 일치도와
   방향 일치도를 산출합니다(`tab09_extractor_stability`). 08번은 이 값을
   **측정 오차의 하한**으로 삼아, 관측된 반복 기준선과 나란히 보고합니다
   (`tab37_baseline_versus_measurement_error`).

두 번째가 중요합니다. 관측된 기준선이 추출기 자체 불안정성과 비슷한 수준이면,
그 변동을 생성기 탓으로 돌릴 수 없습니다. 08번이 이를 자동으로 경고합니다.

---

## 규약

**그림**
- seaborn 회색조, 캡션 없음
- PNG와 PDF 동시 저장, 600 dpi
- 파일명 `fig{번호}_{내용}`
- 모형 구분은 색이 아니라 선 종류로 (흑백 인쇄 대응)

**표**
- CSV, `utf-8-sig`
- 파일명 `tab{번호}_{내용}`

**캐시**
- LLM 호출 결과는 조건별로 파일 저장, 있으면 건너뜀
- 프롬프트를 고치면 해시가 바뀌어 새로 생성됨
- 재생성하려면 해당 블록 디렉터리를 지웁니다

**재현성**
- 시드 42 고정
- 모형·LLM 버전, 온도, 실행 시각을 `logs/`에 기록

---

## 실험 블록

| 블록 | 무엇을 바꾸는가 | 생성 수 |
|---|---|---:|
| A | 예측모형 × XAI 방법 | 5,400 |
| A' | A와 동일 (이탈 도메인) | 5,400 |
| B | LLM × 프롬프트 | 10,800 |
| C | 두 효과 직접 비교 | 4,800 |
| D | 아무것도 안 바꿈 (τ=1.0) | 3,600 |
| D-0 | 아무것도 안 바꿈 (τ=0) | 3,600 |
| | | **33,600** |

07은 D → D-0 → A → A' → B → C 순으로 실행합니다. 가장 싼 블록을 먼저 돌려
파이프라인 문제를 조기에 발견하기 위해서입니다.

---

## 분석의 핵심

**기준선**: 조건을 하나도 바꾸지 않고 다시 생성했을 때의 내용 거리.

**추가 변동(Δ)**: 조건을 하나 바꿨을 때 기준선보다 얼마나 더 벌어지는가.

```
ΔD = D(조건 변경) − D(반복 기준선)
```

Δ는 **사례 단위로 계산한 뒤 분포를 봅니다.** 전체 평균끼리 빼지 않습니다.
신뢰구간은 사례를 재표집 단위로 하는 cluster bootstrap으로 구합니다.

이 구조 덕분에 "몇 %면 합격"이라는 절대 기준이 필요 없습니다. 결론이 상대적
비교이기 때문입니다.

---

## 주의사항

- **추출기의 타당도가 연구의 기반입니다.** 05의 κ가 기준에 못 미치면 진행하지
  마십시오.
- **추출기 자체 불안정성이 기준선의 바닥입니다.** 08의 `tab37`을 확인하고, 비율이
  2 미만이면 생성기 기여를 분리할 수 없다고 명시하십시오.
- **공통 변수가 없어 거리가 정의되지 않는 사례**는 자동 제외되는데, 이들이 가장
  심하게 갈라진 사례입니다. 결측률을 거리 평균과 함께 보고하십시오.
- **음수 Δ는 오류가 아닙니다.** 제약 프롬프트처럼 변동을 억제하는 조건에서
  나타납니다. 잘라내지 말고 비율을 보고하십시오.
- **대조 도메인은 A블록에만 적용됩니다.** LLM·프롬프트 효과의 도메인 일반화는
  주장할 수 없습니다.
- 10은 보조 분석입니다. 수렴에 실패하면 실패했다고 적습니다. 09가 주 결과입니다.
- 논문에서 **타당도와 신뢰도를 구분**하십시오. 05는 "사람처럼 읽는가"(타당도),
  04는 "같은 글을 두 번 같게 읽는가"(신뢰도)를 잽니다.
