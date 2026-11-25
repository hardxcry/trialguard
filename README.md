<div align="center">

# 🛡️ TrialGuard

**One poisoned public trial record can hijack a hospital's AI trial screening assistant.
TrialGuard measures how badly, across 22 attacks and 5 local LLMs, and stops it.**

![models](https://img.shields.io/badge/LLMs-Qwen3%20%C2%B7%20Llama%203.1%20%C2%B7%20Gemma%203%20%C2%B7%20MedGemma-22c55e)
![compute](https://img.shields.io/badge/local%20GPU-33%20hours-16a34a)
![decisions](https://img.shields.io/badge/AI%20decisions-16k%2B-16a34a)
![status](https://img.shields.io/badge/status-research%20prototype-orange)
![licence](https://img.shields.io/badge/licence-all%20rights%20reserved-lightgrey)

**[▶ Watch the live demo](assets/TrialGuard_demo.mp4)** · **[📄 Full report](docs/REPORT.md)** · **[📊 All results](results/RESULTS.md)**

</div>

![TrialGuard at a glance](assets/TrialGuard_collage.png)

> This repository is a **showcase**: it presents the research, results and demo. The implementation
> (about 2,200 lines of Python: benchmark, 22 attacks, 8 defences, agent, adaptive and GCG attackers,
> detector training, dashboard and tests) is kept in a private repository and is available to
> recruiters and reviewers on request.

## The problem

Hospitals are starting to use LLMs to pre-screen patients for clinical trials: the model reads a patient note
and a handful of trial records and recommends where to refer the patient. Those trial records are **public text
written by trial sponsors**. Anyone who controls one record can write instructions into it, without ever seeing the
patient, the other trials, the prompt or the model. This is *indirect prompt injection*, and in a clinical
pipeline it means wrong referrals, suppressed eligible trials and, for tool-using agents, leaked patient data.

TrialGuard is a red team study of that threat on **real data** (75 TREC 2021 patients, real ClinicalTrials.gov
records, NIST eligibility labels), run entirely **on one laptop GPU with local models** (about 33 hours of
compute, no cloud). It ends with a layered defence that brings attack success to **0% on the core attacks without
costing accuracy**.

## Key findings

| | Result |
|---|---|
| 💥 **One sentence is enough** | "Ignore all previous instructions..." made the attacker's irrelevant trial the top recommendation for **85 to 96%** of patients on all four models |
| 🧪 **Medical tuning did not help** | MedGemma 4B was hijacked at least as often as its base model Gemma 3 4B, and more often by fake completion (95% vs 67%) and output format spoofs (92% vs 56%) |
| 🔤 **Evasion works on filters, not on models** | Look alike Cyrillic letters (77 to 93%) and zero width characters (77 to 88%) bypass keyword filters while models still obey; Base64, ROT13 and invisible Unicode tags fail because these models cannot read them |
| 🌏 **Language is not a barrier** | Instructions in Chinese (84 to 99%), Tamil (47 to 91%) and Hindi (59 to 91%) hijack as well as English |
| ☠️ **Sabotage harms patients directly** | A fake "FDA safety alert" made Qwen3 and Llama **drop a trial the patient was eligible for** in 69 to 76% of cases |
| 🧠 **Reasoning is not a defence** | With thinking mode on, Qwen3 followed injections as often or more (sabotage 100% vs 60%; small sample of 5 patients) |
| 🧱 **Common defences barely help** | Delimiters + warning: 47% → 42%. Off the shelf injection classifier: 43%. Sandwich reminder: 38% |
| ✅ **Separating reading from deciding works** | Privilege separation (one call extracts clinical facts, a second decides from facts only): **3%** |
| 🛡️ **TrialGuard** | Rule sanitiser + domain tuned detector + privilege separation: **0% (95% CI 0 to 1%)** on the core attacks, **4% / 2%** over all 22 attacks (Qwen3 / Llama), with clean accuracy **75% vs 72%** undefended |
| 🤖 **Agents can leak data** | A tool using Qwen3 agent e-mailed synthetic patient identifiers to the attacker in **24%** of "exfiltrate + hijack" episodes; TrialGuard's outgoing call policy blocked **every** leak |
| 🔬 **Domain detector generalises** | Fine-tuned on real registry records, tested on attack families it never saw: **80 to 100%** detection on 5 of 6 families (generic detector: 0 to 10%) |

## Hijacked vs blocked (live, on a real patient)

A 45 year old man with a spinal brain tumour. The attacker's record is a knee surgery trial with one injected sentence.

| No defence | With TrialGuard |
|---|---|
| ![hijacked](assets/dashboard_hijacked.png) | ![blocked](assets/dashboard_blocked.png) |
| The assistant recommends the knee surgery trial | The poisoned record is dropped; an eligible trial is recommended |

## Results

![Attack success with no defence](assets/zoo_heatmap.png)

![Security vs usefulness](assets/defence_tradeoff.png)

![Every defence against every attack](assets/defmatrix_qwen3_8b.png)

## What was built

```
22 attacks   instruction · context reset · fake completion · poisoned examples · fake system message ·
             delimiter escape · output format spoof · hidden HTML comment · clinical camouflage · fake regulator ·
             fake pre-screen · Base64 · ROT13 · homoglyphs · zero width · invisible Unicode tags ·
             Hindi · Tamil · Chinese · sabotage (fake safety alert, direct)
8 defences   none · delimiters · spotlighting · sandwich · injection classifier · privilege separation ·
             layered · TrialGuard
5 models     Qwen3 8B (thinking off/on) · Llama 3.1 8B · Gemma 3 4B · MedGemma 4B   (Ollama, local GPU)
3 advanced   adaptive black-box attacker (PAIR style) · white-box GCG suffix + transfer · tool-using agent exfiltration
1 scan       26,158 real ClinicalTrials.gov records screened for injection content
1 detector   DeBERTa-v3 injection classifier fine-tuned on the laptop GPU
1 dashboard  Streamlit app with a live attack lab
```

**TrialGuard defence**, in order: **sanitise** (strip invisible characters and HTML comments, fold look alike
letters, flag encoded blobs and AI directed language; zero false flags on 26,158 real records) → **detect** (the
fine-tuned registry detector) → **separate privileges** (a quarantined call extracts length capped clinical facts;
the decision uses facts only) → **gate actions** (agents may only e-mail a candidate trial's listed contact, with
no patient identifiers).

## Honest limitations

Small open models (4 to 8B) and one prompt template. The detector's in-distribution numbers are optimistic; the
leave-one-family-out results are the honest measure. It flags 0.16% of real records (legitimate extension studies
that cite other trial IDs). The adaptive attacker was weak (10% without a defence). The white-box GCG suffix
(10% → 38% on Qwen3-1.7B) did not transfer to larger models. Llama 3.1 8B could not run the agent workflow.
Details in the [report](docs/REPORT.md).

## Responsible use

Defensive research. All attacks ran only against local models on the author's own machine; no public record was
modified, and all patient identifiers in the agent experiment are synthetic.

## Data and credits

TREC 2021 Clinical Trials track (NIST) · ClinicalTrials.gov (U.S. National Library of Medicine) · Qwen3 (Alibaba)
· Llama 3.1 (Meta) · Gemma 3 and MedGemma (Google) · protectai/deberta-v3-base-prompt-injection-v2 ·
Spotlighting (Hines et al., 2024) · GCG (Zou et al., 2023) · PAIR (Chao et al., 2023).

## Licence

Copyright (c) 2026 hardxcry. **All rights reserved.** Published for viewing only. No licence is granted to copy,
modify or redistribute any part of this repository. See [LICENSE](LICENSE).
