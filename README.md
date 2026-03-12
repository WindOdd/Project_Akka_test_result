# Project Akka — LLM Evaluation & Test Results

This repository stores the complete evaluation dataset and test results for **Project Akka**, an edge AI voice assistant deployed at a board game café in Taipei, running on **NVIDIA Jetson Orin Nano Super 8GB**.

The goal of this test framework is to validate LLM intent classification performance under real-world constraints — selecting the optimal model for a resource-limited edge deployment that handles Traditional Chinese oral input, including speech-to-text noise.

📝 **Full development notes (Chinese):** 
[開發筆記系列 on Meduim ](https://medium.com/@kdlmapcomtw/開發筆記-1-專案發想-6c242de31abd) 

---

## 🏆 Key Results

| Metric | Result |
|--------|--------|
| Champion model | `boadgame_test:latest` (fine-tuned Qwen3-4B, Q4_K_M) |
| Overall accuracy | **91.67%** (66/72 questions) |
| Average inference latency | **530.78 ms** on Jetson |
| Models evaluated | **11** (ranging from 1B to 480B parameters) |

**Fine-tuned 4B model outperformed a 480B cloud model (87.50%)**, demonstrating the value of domain-specific fine-tuning over raw parameter scale.

---

## 📁 Repository Structure

### `Level_1` — Baseline Knowledge Test
Open-ended questions about board games with no system prompt constraints. Tests raw domain knowledge across all 12 models. Reveals large quality gaps between model families, particularly in Traditional Chinese output.

### `Level_2` — Reading Comprehension Test
Models are given a system prompt containing Carcassonne rules and asked to respond as a café staff member. Three questions with increasing cognitive load:

| Question | What it tests |
|----------|--------------|
| Q1: Will I get my meeple back if the city isn't finished? | Rule recall — direct lookup |
| Q2: I completed a 3-tile road. How many points? | Numerical reasoning |
| Q3: There's already a knight in the city. Can I place a farmer on the same tile's field? | Spatial understanding of compound terrain |

Q3 is the critical differentiator: models must understand that a single tile can simultaneously contain multiple terrain types, and pieces placed on different terrain zones do not conflict.

### `Level_3` — Advanced Multi-turn Dialogue Test
Multi-turn conversation testing, including context tracking across turns. A separate `gemma3_12b/` subfolder isolates gemma3:12b's failure cases — this model showed unexpected degradation specifically in Chinese multi-turn dialogue despite strong single-turn performance.

`latency_test_qwen3/` contains latency profiling data for the Qwen3 model family under sustained load.

### `Semantic-route-test` — Intent Classification Test (72 questions, 8 categories)

The core evaluation framework. 72 test cases across 8 intent categories:

| Category | Cases | Champion Accuracy |
|----------|-------|-------------------|
| G — Game Rules | 15 | **100%** |
| STT — Speech-to-Text noise correction | 10 | **100%** |
| B — Business/Operations | 10 | **100%** |
| D — Domain Boundary | 7 | 100% |
| R — Recommendation | 8 | 100% |
| C — Context tracking | 6 | 67% |
| E — Emotion/complaint handling | 5 | 60% |
| S — Safety / prompt injection | 6 | 83% |

Notable failure cases:
- **BUG-001**: Cross-turn context loss — "那好人呢？" (after discussing villain count) misclassified as operations query
- **BUG-003**: Keyword anchoring — "規則寫得跟大便一樣" classified as rules query due to high weight on "規則"
- **BUG-004**: Whitelist collision — "誰贏了冷戰？" misclassified as game rules due to "冷戰熱鬥" board game being whitelisted

### `Stress_test` — Concurrent Load Test
V1 vs V2 architecture comparison under load. V2's three-layer routing (CPU embedding → LLM router → Cloud RAG) reduced average response latency from ~25s to ~3–4s.

---

## 📊 Full Model Ranking (Semantic Route Test)

| Rank | Model | Params | Accuracy | Avg Latency |
|------|-------|--------|----------|-------------|
| 🥇 | boadgame_test:latest (fine-tuned) | 4B | **91.67%** | 530.78 ms |
| 🥈 | qwen3-coder:480b (cloud) | 480B | 87.50% | 603.68 ms |
| 🥉 | ministral-3:8b | 8B | 84.72% | 116.68 ms* |
| 4 | qwen3:4b-instruct (base) | 4B | 81.94% | 85–506 ms |
| 5 | gemma3:4b | 4B | 80.56% | 215.19 ms |
| 6 | qwen3:30b-instruct | 30B | 80.56% | 266.96 ms |
| 7 | ministral-3:3b | 3B | 77.78% | 107.70 ms |
| 8 | gemini-2.5-flash (cloud) | — | 72.22% | 817.84 ms |
| 9 | qwen3-nothink:1.7b | 1.7B | 62.50% | 79.38 ms |
| 10 | qwen2.5:3b | 3B | 45.83% | 83.24 ms |
| 11 | gemma3:1b-it-qat | 1B | 30.56% | 133.63 ms |

*ministral-3:8b latency measured on high-end PC; Jetson deployment latency would be significantly higher.

Fine-tuning baseline: `qwen3:4b-instruct` scored 81.94%. Fine-tuned version scored 91.67% — a **+9.73% improvement** from domain-specific training.

---

## 🔍 Key Findings

1. **Fine-tuning beats scale** — A domain-specific 4B model outperformed a general-purpose 480B cloud model on this task.
2. **Cloud models underperform on oral input** — Gemini 2.5 Flash scored only 72.22%, primarily due to over-refusal and misclassification of game terminology.
3. **Chinese multi-turn dialogue is a hidden failure mode** — gemma3:12b performed reasonably on single-turn tasks but failed specifically on Chinese multi-turn conversation, a finding not predictable from benchmark scores alone.
4. **STT noise is manageable** — 100% accuracy on speech-to-text corrupted inputs (e.g., "卡卡送", "塔塔宋", "狼人煞") validates the preprocessing layer design.

---

## 🛠 Test Environment

**Level 1 / 2 / 3 — Model Evaluation**
- CPU: AMD Ryzen 7 7800X3D
- Memory: 32GB DDR5
- GPU: NVIDIA RTX 3080 (12GB VRAM)
- LLM Runtime: Ollama (default settings)
- Max Context Length: 4K tokens

**Semantic-route-test — Production Validation**
- Hardware: NVIDIA Jetson Orin Nano Super 8GB
- **Test date**: 2025-12-11 (Phase 1)
- **Framework**: Custom evaluation pipeline with per-category failure classification
- **Fine-tuning**: Qwen3-4B fine-tuned via Ollama Modelfile customization