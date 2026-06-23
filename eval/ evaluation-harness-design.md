# Evaluation Harness 설계 문서
 
## 개념 정의
 
| 구분 | 개념 및 정의 | 이 프로젝트에서의 예시 |
|---|---|---|
| **Evaluation Harness** | 평가를 **실행하는 테스트 환경이자 인프라** 그 자체 | 데이터셋을 읽어 게이트웨이에 쏘고, 성능을 수집·기록·시각화하는 **자동화 파이프라인 코드** |
| **Evaluation Benchmarks** | 모델의 능력을 비교하기 위해 규격화된 **표준 문제집 또는 데이터셋** | Easy/Medium/Hard 분류가 완료된 **300개 이상의 프롬프트 셋** (public dataset 기반) |
| **Evaluation Metrics** | 결과물이 얼마나 훌륭한지 평가하는 **점수 및 측정 기준** | Cost($), p95 Latency(ms), Cache Hit Rate(%), Routing Accuracy, Judge Score |
 
---
 
## 1. Measurement Target
 
### Cost
- Query 당 비용 (input/output 토큰 × 모델별 단가)
- 100개 쿼리당 총 비용
- Baseline 대비 절감율
  - 정의: 게이트웨이 없이 전부 최고 모델로 보냈을 때의 비용에서, 현재 게이트웨이가 실제 지출한 비용을 뺀 값
  - 수식: `Cost(baseline) - Cost(Gateway)`
### Latency
- p50, p95, p99 지연시간
- Semantic Cache Hit Rate
  - 정의: 전체 요청 중 LLM을 호출하지 않고 Qdrant 캐시에서 바로 응답한 비율
  - 수식: `(Cache Hits / Total Requests) × 100`
- Cache hit 시 latency vs cache miss 시 latency 분리해서 측정
### Quality
- **라우팅 정확도 (Routing Accuracy)**
  - 정의: 정답셋에 지정해 둔 '가장 가성비 좋은 최적 모델'(oracle)과 실제 라우터가 선택한 모델이 얼마나 일치하는가
  - 라벨링 방법은 5번 섹션 참조 (Oracle Sweep)
  - 단순 일치율만으로는 부족하므로 아래 두 가지로 세분화:
    - **Over-provisioning Rate**: 라우터가 oracle보다 비싼 모델을 고른 비율 (비용 손해, quality는 괜찮음)
    - **Under-provisioning Rate**: 라우터가 oracle보다 싼 모델을 골랐는데 quality threshold 미달인 비율 (quality 손해, 더 치명적)
- **응답 품질 유지율 (Quality Score)**
  - 정의: 경량 모델(Haiku 등)로 라우팅된 응답이 프론티어 모델(GPT-5) 응답 품질 대비 몇 점인가 (Judge가 1~5점 척도로 채점)
- **정답 정확도** (정답이 있는 query인 경우)
  - exact match (GSM8K) 또는 unit test pass/fail (HumanEval/MBPP)
- **Judge Score** (정답 없는 open-ended query인 경우)
  - rubric 기반 1~5점, DeepEval의 GEval 등으로 측정 (4번 섹션 참조)
- **Escalation Rate**
  - 정의: 경량 모델이 답을 잘못해서 Judge Agent에 의해 상위 모델로 재요청(escalation)된 비율
  - 이 비율이 너무 높으면 라우터(Strategy 2/3) 자체를 수정해야 한다는 신호
---
 
## 2. Benchmark Dataset
 
### 난이도 계층 구조 — 균등 분포로 디자인
 
```
Easy   (~33%): 사실 질문, 정의, 간단한 변환
               예: "파이썬에서 리스트 reverse 하는 법?"
 
Medium (~33%): 멀티스텝 추론, 코드 디버깅, 요약
               예: "이 함수의 버그를 찾고 고쳐줘" (10-20줄 코드 포함)
 
Hard   (~34%): 복잡한 추론, 긴 컨텍스트, 수학적 증명,
               ambiguous한 요구사항
               예: "이 시스템 아키텍처의 trade-off를 분석하고
               대안을 제시해줘"
```
 
**왜 균등 분포인가**: 이 프로젝트의 핵심 스토리는 "라우터가 난이도별로 얼마나 정확하게 모델을 고르는가"이다. 실제 production 트래픽처럼 easy 비중을 높게(60% 등) 가져가면 라우터가 hard 쿼리에서 어떻게 행동하는지 충분히 검증되지 않은 채 전체 accuracy 숫자가 좋게만 보일 위험이 있다. 난이도별 라우터 성능을 동일한 비중으로 비교하기 위해 33/33/34로 설계한다.
 
**난이도 기준이 모호할 때**: 실제 유저 쿼리를 정제해 난이도별로 라벨링해 둔 오픈소스 데이터셋인 **WildBench (V2)**의 태스크 예시들을 참고해 기준을 보정한다.
- Reference: https://arxiv.org/html/2406.04770v2
### 쿼리 소스 — Public 데이터셋만 사용
 
LLM 합성 생성은 사용하지 않는다. 이유:
- Ground truth가 없어 신뢰도 검증이 추가로 필요함
- 난이도 라벨도 LLM이 매긴 것이라 검증 부담이 큼
- "내가 만든 데이터로 내가 평가했다"는 순환논리 의심을 받을 수 있음
- Public benchmark를 쓰면 이미 검증된 표준이라는 신뢰도를 얻을 수 있고, 작업 시간도 더 짧음
**도메인은 코드/기술 도메인 1개에 집중**한다 (Q4 논의 결과). 다양한 도메인을 섞으면 도메인마다 난이도 기준이 달라져 비교가 흐려짐
 
| 난이도 | 데이터셋 | 비고 |
|---|---|---|
| Easy | Natural Questions / TriviaQA (샘플링) | 짧은 정답 → exact/fuzzy match |
| Easy~Medium | HumanEval, MBPP | unit test로 ground truth 자동 검증, 난이도 태그 이미 존재 |
| Medium~Hard | GSM8K | 숫자 정답 → exact match, 멀티스텝 추론 |
 
이 세 데이터셋만 조합해도 ground truth가 있는 쿼리 300개 이상을 확보할 수 있고, 난이도 분포도 데이터셋 특성상 자연스럽게 균등에 가깝게 갈린다.
 
**정답 없는 open-ended 쿼리 보충**: 위 데이터셋들은 대부분 ground truth가 명확한 쿼리다. 요약/비교/계획 수립 같은 open-ended 쿼리는 ShareGPT에서 일부 샘플링하여 보충하고, 이 부분만 Judge(DeepEval) 평가로 처리한다.
 
**정답 유무 비율**: 정답 있음 50% / 정답 없음 50%
- 정답 있는 쪽: routing accuracy와 quality 측정의 anchor 역할
- 정답 없는 쪽: 실제 쿼리 다양성을 보여주는 역할, Judge 점수로만 평가
### 분류 축: Task / Level (2축)
 
- **Task**: factual / debugging / summarization / math_reasoning / system_design 등
- **Level**: easy / medium / hard
 
---
 
## 3. Dataset Schema
 
```json
{
  "id": "q_0042",
  "query": "What's the time complexity of merge sort?",
  "source_dataset": "natural_questions",   // 출처 데이터셋 (natural_questions / humaneval / mbpp / gsm8k / sharegpt)
  "difficulty_label": "easy",               // 사전 라벨링된 난이도 (oracle 검증 대상)
  "task": "factual",                        // factual / debugging / summarization / math_reasoning / system_design
  "ground_truth": "O(n log n)",             // 있으면 채우고, 없으면 null
  "eval_method": "exact_match"              // exact_match / unit_test / judge_only
}
```
 
JSONL로 저장. 200~500개면 충분하다. 처음엔 100개로 파이프라인을 검증하고, 검증 후 전체 규모로 확장한다.
 
---
 
## 4. Judge 설계 (정답 없는 쿼리용)
 
### 두 종류의 Judge 구분
 
| | Judge #1 (Evaluation Harness용) | Judge #2 (게이트웨이 내부 Judge Agent) |
|---|---|---|
| 목적 | eval 결과를 채점하기 위한 것 | 실시간으로 escalation 필요 여부 판단 |
| 구현 단계 | Step 0 — harness 설계 시점 | Step 4 — Agentic Router 구현 시점 |
| 검증 기준 | ground truth로 sanity check | Judge #1을 기준으로 검증 |
 
Judge #1이 먼저 신뢰성을 확보해야, 그걸 기준으로 Judge #2(production 컴포넌트)를 검증할 수 있다. 순서를 거꾸로 하면 "뭘로 뭘 검증하는지"가 순환논리가 되어버린다.
 
### 평가 도구 적용 범위
 
**ground truth가 있는 쿼리에는 DeepEval/LLM judge를 쓰지 않는다.**
 
| 쿼리 타입 | 평가 방법 |
|---|---|
| 코드 (HumanEval/MBPP) | unit test 실행 (pytest pass/fail) |
| 수학 (GSM8K) | 숫자 exact match |
| 일반 factual (Natural Questions) | exact/fuzzy match |
| open-ended (ShareGPT, 요약/비교) | **DeepEval (GEval 커스텀 rubric)** |
 
이미 binary하게 정확히 검증 가능한 쿼리에 LLM judge를 또 돌리는 것은 불필요한 비용과 노이즈를 추가할 뿐이다. DeepEval은 ground truth가 없는 부분에만 선택적으로 적용한다.
 
`LM-Evaluation-Harness`는 모델 자체를 표준 벤치마크로 평가하는 도구라 스코프가 다르므로 사용하지 않는다. `OpenAI Evals`보다 `DeepEval`을 택한 이유는 provider-agnostic이라 Anthropic/OpenAI 모델을 함께 쓰는 이 프로젝트에 더 적합하기 때문이다.
 
### Judge Prompt (GEval 커스텀 rubric)
 
```python
judge_prompt = """
You are evaluating an AI assistant's response.
 
Query: {query}
Response: {response}
 
Rate the response on a scale of 1-5 based on:
- Correctness (factually accurate?)
- Completeness (fully addresses the query?)
- Clarity (well-structured, no confusion?)
 
Output ONLY a JSON object:
{{"score": <1-5>, "reasoning": "<one sentence>"}}
"""
```
 
Judge 호출(GPT-5/Opus 등 강한 모델 사용 시) 비용은 "evaluation cost"로 production 비용과 분리해서 집계한다.
 
### Judge 신뢰도 검증 (Step 0에서 수행)
 
ground truth가 있는 쿼리(GSM8K, HumanEval) 일부에 대해 일부러 DeepEval judge도 같이 돌려서, judge 점수와 실제 pass/fail이 얼마나 일치하는지 확인한다. 예를 들어 실제로 unit test를 통과한 코드인데 judge가 낮은 점수를 줬다면 rubric을 수정해야 한다.
 
같은 응답을 3번 채점해서 점수 분산이 크면 rubric을 더 구체화한다. 이 검증을 마친 뒤에야 ground truth가 없는 open-ended 쿼리의 judge 점수를 신뢰할 수 있다.
 
Step 4에서 구현하는 Judge Agent(Judge #2)는 이렇게 신뢰도가 확보된 Judge #1 기준으로 다시 한번 검증한다.
 
---
 
## 5. Routing Accuracy — Oracle Label 생성 방법
 
"가장 가성비 좋은 모델"이라는 라벨에는 절대적 정답이 없다. 사람이 주관적으로 미리 판단하는 대신, 데이터로 oracle label을 만든다.
 
### Oracle Sweep
 
1. 모든 쿼리를 **모든 후보 모델**(예: Haiku, GPT-4o-mini, GPT-5)로 다 실행
2. 각 모델 응답의 quality score 측정 (정답 있으면 exact match/unit test, 없으면 Judge #1)
3. "quality threshold(예: 4/5점) 이상을 넘긴 모델 중 가장 저렴한 모델"을 oracle 정답으로 라벨링
```python
def determine_oracle_model(query, results_by_model, quality_threshold=4.0):
    # results_by_model = {"haiku": {"cost": 0.001, "quality": 4.2},
    #                      "gpt-4o-mini": {"cost": 0.003, "quality": 4.6}, ...}
    candidates = [m for m, r in results_by_model.items() if r["quality"] >= quality_threshold]
    if not candidates:
        return max(results_by_model, key=lambda m: results_by_model[m]["quality"])  # 품질 기준 fallback
    return min(candidates, key=lambda m: results_by_model[m]["cost"])
```
 
장점: 사람의 주관이 들어가지 않아 일관성 있고 재현 가능하다. 단점: 쿼리 × 모델 조합을 다 실행해야 해서 비용이 모델 개수만큼 늘어나지만, 이는 eval set 생성 시 한 번만 드는 일회성 비용이므로 production 비용과 분리해서 생각한다.
 
### 사람 검증 (소규모 샘플)
 
전체 데이터셋을 사람이 라벨링하는 것은 비현실적이지만, oracle sweep으로 만든 라벨 중 50~100개를 샘플링해 직접 검증한다. 검증 기준:
- "이 쿼리에 Haiku 응답이면 충분히 정확하고 완결되는가?" → Yes면 easy 라벨 확정
- "GPT-4o-mini는 정확한데 Haiku는 틀렸는가?" → medium 확정
- "GPT-5만 정확한가?" → hard 확정
목적은 oracle sweep 방식 자체가 신뢰할 만한지 확인하는 것이다 (4번 섹션의 Judge 신뢰도 검증과 같은 논리).
 
### Routing Accuracy 계산
 
```
Routing Accuracy = (라우터가 선택한 모델 == oracle 모델인 쿼리 수) / 전체 쿼리 수
 
Over-provisioning Rate  = 라우터가 oracle보다 비싼 모델을 고른 비율
Under-provisioning Rate = 라우터가 oracle보다 싼 모델을 골랐는데 quality threshold 미달인 비율
```
 
단순 accuracy만 보고하지 않고 두 비율을 나눠서 리포트하면 "내 라우터는 over-provisioning 5%, under-provisioning 2%로 안전한 방향으로 편향되어 있다" 같은 분석이 가능해지고, 이는 단순 accuracy 숫자보다 인사이트가 크다.
 
---
 
## 6. 측정 자동화 스크립트 구조
 
```
eval/
├── dataset/
│   └── queries.jsonl          # Step 3의 스키마
├── oracle/
│   └── build_oracle_labels.py # Step 5의 Oracle Sweep 실행
├── runners/
│   └── run_eval.py            # 쿼리셋을 게이트웨이에 던지고 결과 수집
├── metrics/
│   └── compute_metrics.py     # cost, latency, quality, routing accuracy 집계
└── results/
    └── strategy_comparison.csv # 최종 비교 테이블
```
 
`run_eval.py`가 할 일:
1. 쿼리셋 로드
2. 각 쿼리를 게이트웨이 엔드포인트로 전송 (전략별로 — baseline, cache, rule-router, llm-router, agent-router)
3. 응답 + 메타데이터(latency, tokens, model_used, cache_hit) 기록
4. ground truth 있는 쿼리는 exact match/unit test, 없는 쿼리는 Judge #1(DeepEval) 호출
5. PostgreSQL 또는 CSV에 저장
---
 
## 7. 최종 비교 테이블 (예시)
 
| Strategy | Avg Cost/100q | p50 Latency | p95 Latency | Quality Score | Cache Hit Rate | Routing Accuracy |
|---|---|---|---|---|---|---|
| Baseline (GPT-5 only) | $X.XX | Xms | Xms | X.X/5 | 0% | - |
| + Semantic Cache | $X.XX | Xms | Xms | X.X/5 | XX% | - |
| Rule-based Router | $X.XX | Xms | Xms | X.X/5 | XX% | XX% |
| LLM Router | $X.XX | Xms | Xms | X.X/5 | XX% | XX% |
| Agentic Router + Judge | $X.XX | Xms | Xms | X.X/5 | XX% | XX% |
 
이 테이블이 프로젝트 README의 핵심 결과물이며, 면접에서 가장 먼저 검토될 부분이다.