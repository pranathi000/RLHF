# Complete Post-Training Pipeline on UltraFeedback

A complete RLHF-style post-training pipeline built on GPT-2 Small using UltraFeedback. Covers SFT, reward modeling, GRPO with KL regularization, KL ablation, and DPO — all on the same dataset with no format mismatches. Built and run on a single T4 GPU in Google Colab.

## What This Project Does

Takes a raw GPT-2 model through every stage of modern post-training:
- SFT on 2000 high-quality chosen responses
- Reward model trained from scratch using Bradley-Terry loss on 1000 preference pairs
- GRPO training with KL regularization and reward hacking analysis
- KL ablation experiment (beta=0.1 vs beta=0.0)
- DPO on the same 1000 preference pairs as alternative to RL

## Stack

| Component | Detail |
|---|---|
| Base model | GPT-2 Small (124M) |
| Reward model | DistilBERT + Linear(768→1) |
| Dataset | UltraFeedback (openbmb) |
| Library | HuggingFace TRL 1.6.0 |
| Hardware | T4 GPU, 14.56 GB VRAM |

## Key Results

| Stage | Key Metric | Result |
|---|---|---|
| SFT | Val loss (epoch 3) | 2.304 |
| Reward model | Val accuracy | 58.7% |
| GRPO vs SFT | Mean reward improvement | +0.19 |
| Reward hacking | "sure" phrase frequency | 5x increase |
| KL ablation | beta=0.0 reward vs beta=0.1 | Worse (-2.575 vs -2.305) |
| DPO | Rewards margin (epoch 2) | 0.688 |

## Key Findings

**Finding 1 — Reward model bias matters more than accuracy.** 56.5% overall accuracy hid a critical failure: 49% accuracy on medium-difficulty pairs. The model learned confident surface tone over actual correctness.

**Finding 2 — GRPO exploited reward model biases.** Stage 4 predicted phrase bias. Stage 5 confirmed it. "sure" appeared 5x more in GRPO outputs while human quality degraded. Reward improved slightly (+0.19) — textbook reward hacking.

**Finding 3 — KL serves two purposes.** Removing KL did not increase reward — it destabilized training (loss spike 50.58 at step 30). KL prevents policy drift AND stabilizes optimization against noisy reward signal.

**Finding 4 — DPO is immune to reward hacking by design.** No proxy reward to exploit. Rewards margin grew from 0.484 to 0.688 across epochs. Fast convergence — train loss dropped 57% from epoch 1 to 2.

## Data Splits

| Split | Size | Purpose |
|---|---|---|
| SFT data | 2,000 | Chosen responses only |
| Preference pairs | 1,000 | Reward model + DPO |
| Eval set | 200 | Held out, zero overlap |

Filtered from 63,967 UltraFeedback examples to 35,002 strong pairs (score gap > 2.0). Zero overlap between all splits.


## The One Insight

Every problem in this pipeline traces to one thing — the reward model is an imperfect proxy. GRPO amplified its biases. Removing KL destabilized training. DPO sidesteps the problem entirely. This is why verifiable rewards (code passes or fails, math is correct or wrong) eliminate reward hacking at the source.


