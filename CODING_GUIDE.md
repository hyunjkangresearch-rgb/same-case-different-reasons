# 부호화 지침서 (Coding Guide)

이 문서는 `data/manual_coding_sheet.csv`를 채우는 분을 위한 것입니다.

코드를 작성하는 일이 아닙니다. **설명 문장을 읽고 표에 옮겨 적는 작업**입니다.
연구방법론에서 이를 coding(부호화)이라 부르기 때문에 열 이름이 `coder1_*`로
되어 있을 뿐입니다.

---

## 1. 왜 이 작업이 필요한가

이 연구는 LLM이 쓴 설명에서 「어떤 변수를 어떤 방향으로 말했는지」를 자동으로
뽑아내는 도구를 씁니다. 그 도구가 제대로 읽는지 확인하려면, **도구와 무관한
기준**이 필요합니다. 사람이 직접 읽은 결과가 그 기준입니다.

따라서 두 가지를 지켜 주십시오.

- 자동 추출 결과를 **보지 마십시오**. 시트에 넣지 않았지만, 혹시 접하게 되더라도
  참고하지 마십시오.
- 두 사람이 **서로 상의하지 않고** 각자 채웁니다.

---

## 2. 필요한 배경지식

**없습니다.** 신용평가나 머신러닝을 몰라도 됩니다.

필요한 것은 영어 문장을 읽고, 지침을 일관되게 적용하는 것뿐입니다. 오히려
도메인 지식이 있으면 설명에 없는 내용을 배경지식으로 채워 넣을 위험이 있습니다.
**문장에 적힌 것만** 표시하십시오.

---

## 3. 채울 열

| 열 | 채우는 사람 |
|---|---|
| `explanation` | (읽기만) LLM이 쓴 설명 |
| `allowed_features` | (읽기만) 사용 가능한 변수명 목록 |
| `coder1_features` | 코더1 |
| `coder1_directions` | 코더1 |
| `coder2_features` | 코더2 |
| `coder2_directions` | 코더2 |

두 분이 파일을 각자 복사해 자기 열만 채운 뒤, 마지막에 합치는 방식을 권합니다.

---

## 4. 작성 방법

설명을 읽고, **언급된 변수를 등장 순서대로** 세미콜론(`;`)으로 구분해 적습니다.
방향도 같은 순서로 적습니다.

### 예시

> - Low income is the main reason the risk is high.
> - A short employment history also pushes the risk up.
> - Owning a home works in the applicant's favour.

```
coder1_features   : person_income; person_emp_length; person_home_ownership_OWN
coder1_directions : increases; increases; decreases
```

### 규칙

- 변수명은 `allowed_features` 열에 있는 표기를 **그대로** 씁니다
- 같은 변수를 두 번 적지 않습니다
- 목록에 없는 것을 말했다면 **적지 않습니다**
- 아무것도 언급하지 않았다면 두 열을 비워 두지 말고 각각 `-` 를 적습니다

---

## 5. 방향 판정 — 가장 중요한 부분

여기서 실수가 가장 많이 납니다.

**방향은 변수의 값이 아니라, 그 변수가 평가에 미친 영향입니다.**

> "**낮은** 소득 때문에 위험이 **높아졌다**"

- 소득이 낮다 → 이건 방향이 아닙니다
- 소득이 위험을 **올리는 쪽**으로 작용했다 → `increases`

| 문장 | direction |
|---|---|
| ~때문에 위험이 높아졌다 | `increases` |
| ~가 위험을 낮췄다 | `decreases` |
| ~가 긍정적으로 작용했다 | `decreases` |
| ~가 우려 요인이다 | `increases` |
| ~를 언급했으나 방향이 불명확 | `unclear` |

### 헷갈리는 사례

**사례 1 — 부정 표현**

> "Not having internet service also contributes to the likelihood of leaving."

인터넷 서비스가 **없다**는 것이 이탈 가능성을 **높였다** → `increases`

**사례 2 — 이중 부정처럼 보이는 것**

> "A longer tenure makes the subscriber less likely to leave."

tenure가 이탈을 **낮췄다** → `decreases`

**사례 3 — 방향이 없는 언급**

> "The monthly charges are also a factor."

요인이라고만 했고 올렸는지 내렸는지 안 적음 → `unclear`

---

## 6. 판단이 어려울 때

- **애매하면 `unclear`** 를 쓰십시오. 억지로 판단하지 마십시오
- 문장이 여러 변수를 한꺼번에 말하면, 각각 따로 적습니다
- 도입부의 "이 고객은 이탈 가능성이 높습니다" 같은 문장은 **변수 언급이 아닙니다**.
  적지 않습니다

---

## 7. 작업량

한 건에 1~2분, 전체 2~4시간 정도입니다. 한 번에 몰아서 하기보다 나눠서 하시되,
**중간에 판정 기준을 바꾸지 마십시오.** 일관성이 정확성보다 중요합니다.

판정 기준이 바뀌었다고 느끼시면, 연구자에게 알리고 앞부분을 다시 검토하십시오.

---

## 8. 끝난 뒤

두 분의 결과를 합쳐 `data/manual_coding_sheet.csv`에 저장합니다.

노트북 05를 실행하면 두 분이 갈린 항목만 뽑아 `data/adjudication.csv`를
만듭니다. 그 항목들은 세 번째 사람 또는 두 분이 함께 상의해 결정합니다.
그 전까지는 상의하지 마십시오.
