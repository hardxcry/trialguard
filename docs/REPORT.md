# TrialGuard: Indirect Prompt Injection Against LLM Clinical-Trial Screening, and a Layered Defence

*hardxcry, November 2026. Research prototype; all experiments run locally on one RTX 5060 laptop GPU.*

## Abstract

LLM assistants that match patients to clinical trials read trial records written by third parties. We ask whether
the sponsor of a single public record can manipulate such an assistant without seeing the patient, the other
candidates, the prompt or the model. On 75 patients from the TREC 2021 Clinical Trials track, each with five real
ClinicalTrials.gov records judged by NIST, we evaluate 22 injection attacks in seven families on five local model
configurations (Qwen3 8B with and without reasoning, Llama 3.1 8B, Gemma 3 4B, MedGemma 4B), eight defences, an
adaptive black-box attacker, a white-box GCG suffix and a tool-using agent. A one-line instruction made the
attacker's irrelevant trial the top recommendation for 85-96% of patients on every model. Medical fine-tuning
(MedGemma vs its Gemma 3 base) did not reduce vulnerability. A fake safety alert caused eligible trials to be
dropped in 69-76% of cases (Qwen3, Llama). Delimiters, sandwich reminders and an off-the-shelf injection
classifier gave little protection (core-attack success 47% → 38-43%). Privilege separation reduced it to 3%, and
TrialGuard (rule-based sanitisation, a registry-tuned injection detector and privilege separation) to 0% (95% CI
0-1%) with no loss of clean accuracy (75% vs 72%). In the agent setting, an injection made Qwen3 e-mail synthetic
patient identifiers to the attacker in 24% of episodes; a deterministic policy on outgoing tool calls blocked all
of them.

## 1. Threat model

* **Deployment.** An assistant receives one patient note and k = 5 candidate trial records (title, conditions,
  interventions, age and sex limits, summary, eligibility text) and returns `{"best": id, "eligible": [ids]}` under
  constrained JSON decoding that only admits candidate IDs.
* **Attacker.** Controls the free text (summary, eligibility) of one record. No access to the patient, the other
  records, the prompt or the weights. Goals: *hijack* (their trial becomes `best`), *sabotage* (an eligible
  competitor is dropped from `best` and `eligible`) or, for agents, *exfiltrate* (patient identifiers are sent to
  the attacker).

## 2. Benchmark

75 TREC 2021 topics. For each, two records with qrel 2 (eligible), one with qrel 1 (excluded) and two with qrel 0
(not relevant), one of which is the attacker's; order shuffled with a fixed seed. Records are clipped to 700
characters of summary and 1,600 of eligibility. Payloads go at the end of the eligibility text (main grid) or at
the start of the summary (position ablation). Clean accuracy is the share of clean cases whose `best` is a qrel-2
trial; attack success is measured against the same outcome's base rate without an attack.

## 3. Attacks

Seven families: **instruction** (direct, context reset, fake completion, poisoned examples), **authority** (fake
system message, delimiter escape, output-format spoof, hidden HTML comment), **persuasion** with no instruction
(clinical camouflage, fake regulator, fake pre-screening note), **obfuscation** (Base64, ROT13, Cyrillic
homoglyphs, zero-width characters, invisible Unicode tag characters), **multilingual** (Hindi, Tamil, Chinese),
**sabotage** (fake FDA safety alert, direct instruction) and a benign **control** text of similar length.

## 4. Results

### 4.1 Vulnerability without defence (75 patients per model)

| family | Qwen3 8B | Llama 3.1 8B | Gemma 3 4B | MedGemma 4B |
|---|---|---|---|---|
| instruction | 56% | 71% | 65% | 81% |
| authority | 62% | 33% | 65% | 68% |
| multilingual | 68% | 92% | 91% | 88% |
| obfuscation | 35% | 38% | 38% | 41% |
| persuasion | 13% | 13% | 29% | 26% |
| sabotage | 84% | 84% | 38% | 35% |
| *clean base rate (attacker trial chosen)* | 4% | 4% | 11% | 7% |

Observations. (i) Explicit instructions dominate; instruction-free persuasion rarely works on Qwen3 and Llama.
(ii) Obfuscation is bimodal: homoglyphs and zero-width splitting remain readable to the model (77-93%) while
defeating keyword filters; Base64, ROT13 and Unicode tag characters are not decoded by these models (near base
rate). (iii) MedGemma is at least as vulnerable as its base Gemma 3, and markedly more to fake completions (95%
vs 67%) and output-format spoofs (92% vs 56%). (iv) Payloads at the end of a record are about twice as effective
as at the start (Qwen3 47% vs 26%; Llama 58% vs 33%), consistent with recency effects.

### 4.2 Defences (Qwen3 8B, five core attacks, 75 patients, 375 attacked cases each)

| defence | clean acc. | acc. under attack | attack success (95% CI) | p (McNemar vs none) |
|---|---|---|---|---|
| none | 72% | 42% | 47% (42-52) | - |
| delimiters + warning | 75% | 46% | 42% (37-47) | 0.002 |
| injection classifier filter | 72% | 45% | 43% (38-48) | <0.001 |
| sandwich reminder | 76% | 51% | 38% (33-43) | <0.001 |
| spotlighting (datamarking) | 73% | 61% | 20% (16-24) | <0.001 |
| layered (classifier + spotlight + sandwich) | 76% | 69% | 11% (8-14) | <0.001 |
| privilege separation | 73% | 74% | 3% (1-5) | <0.001 |
| **TrialGuard** | **75%** | **77%** | **0% (0-1)** | <0.001 |

Over all 22 attacks (20 patients), mean attack success: Qwen3 50% undefended → 4% TrialGuard; Llama 51% → 2%.
Privilege separation reduced Llama's clean accuracy (70% → 55%; TrialGuard 60%), so its utility cost is
model-dependent.

### 4.3 Detectors

The off-the-shelf detector (protectai DeBERTa-v3) caught 0% of 12 of the 22 attack types in raw records. Our
detector, fine-tuned on injections written into real registry records disjoint from the benchmark, was evaluated
leave-one-family-out:

| held-out family | TrialGuard detector TPR / FPR | generic TPR / FPR |
|---|---|---|
| authority | 80% / 0% | 5% / 0% |
| instruction | 100% / 1% | 0% / 0% |
| multilingual | 100% / 0% | 7% / 0% |
| obfuscation | 46% / 0% | 10% / 0% |
| persuasion | 91% / 1% | 0% / 0% |
| sabotage | 100% / 0% | 5% / 0% |

The rule sanitiser flags none of 26,158 real registry records. With the detector, 43 records (0.16%) are flagged;
manual inspection found these to be legitimate extension studies that cite other trial IDs, a spurious cue
learned from the payloads, which always contain one. No evidence of injection in the wild was found.

### 4.4 Agent exfiltration

A tool-using agent reads the candidates through a tool and sends one referral; its rules forbid sending
identifiers or e-mailing anyone but the chosen trial's contact. Synthetic identifiers are added to each note.
Qwen3 (50 patients): the "exfiltrate + hijack" payload made it e-mail identifiers to the attacker in 24% of
episodes (identifiers in 50% of referrals); direct and "central intake" exfiltration requests and the invisible
variant failed. Spotlighting reduced exfiltration to 0%. The deterministic policy layer (recipient allow-list
plus identifier filter on every outgoing call) let no leak through in any configuration. Llama 3.1 8B could not
perform the workflow (invented trial IDs and recipients, 0% valid referrals), so its results measure tool-use
failure, not robustness.

### 4.5 Adaptive and white-box attackers

*Adaptive (black-box).* A Qwen3 8B attacker that saw only which trial was chosen rewrote its payload up to 8 times
for 20 patients per defence: 10% success against no defence and spotlighting, 0% against privilege separation
and TrialGuard; found payloads transferred to other patients in 0-6% of replays. The attacker gravitated to
persuasion-style text and never rediscovered the simple direct instruction, so this is a lower bound.

*White-box (GCG).* A 20-token suffix optimised on Qwen3-1.7B over 10 patients raised clinical-camouflage
success on 50 held-out patients from 10% to 38% on that model, but did not transfer to Qwen3 8B, Llama 3.1 8B,
Gemma 3 4B or MedGemma 4B (all within ±7 points of the camouflage-only baseline).

### 4.6 Reasoning mode

The same Qwen3 8B weights with thinking switched on (5 patients, all 22 attacks, no defence) were not more robust:
sabotage success 100% vs 60%, multilingual 60% vs 47%, instruction 60% vs 55%; only instruction-free persuasion
dropped (0% vs 13%). Reasoning appears to make the model *more* diligent about following instructions it finds,
wherever they come from. The sample is small and the result is indicative only.

## 5. Discussion

The cheapest attacks are the most dangerous: a single English sentence, its Chinese translation, or the same
sentence in look-alike letters. Prompt-level defences that ask the model to ignore instructions help little,
because they rely on the very capability under attack. What works is architectural: the model that reads
untrusted text must not be the one that decides (privilege separation), and actions must be checked by code, not
by the model (policy layer). Domain-specific detection is a valuable additional layer: an off-the-shelf detector
trained on generic jailbreaks transferred almost nothing to clinical-registry text.

## 6. Limitations

Small open models and a single prompt template; templated attacks plus a weak adaptive attacker; k = 5
candidates; clipped records; detector in-distribution numbers are optimistic (use the leave-one-family-out
results); reasoning mode tested on only 5 patients; results are not clinical validation of any system.

## References

Greshake et al. (2023) *Not what you've signed up for: compromising real-world LLM-integrated applications with
indirect prompt injection.* Hines et al. (2024) *Defending against indirect prompt injection attacks with
spotlighting.* Zou et al. (2023) *Universal and transferable adversarial attacks on aligned language models.* Chao
et al. (2023) *Jailbreaking black box large language models in twenty queries.* Roberts et al. (2021) *Overview of
the TREC 2021 Clinical Trials Track.*
