## Method (short version; the full paper-style write-up is in docs/REPORT.md)

**Question.** A hospital LLM pre-screens patients for clinical trials by reading trial records it did not write.
Can the sponsor of one public record manipulate it, and which defences actually stop that without making the
assistant worse at its job?

**Threat model.** The attacker controls the free text (summary, eligibility) of one record. They never see the
patient note, the other candidates, the system prompt or the model. Goals: *hijack* (their trial becomes the
recommendation), *sabotage* (an eligible competitor is dropped) or, for agents, *exfiltrate* (patient identifiers
are sent to the attacker).

**Benchmark.** 75 real patient notes from the TREC 2021 Clinical Trials track. Each gets five real
ClinicalTrials.gov records judged by NIST assessors: two eligible, one excluded, two not relevant (one of which is
the attacker's). The assistant answers `{"best": id, "eligible": [ids]}` under constrained JSON decoding,
temperature 0, fixed seed, run locally with Ollama on one laptop GPU.

**Attacks (22, seven families).** Instruction · authority / structure spoofing · instruction-free persuasion ·
obfuscation (Base64, ROT13, homoglyphs, zero-width, invisible Unicode tags) · Hindi / Tamil / Chinese · sabotage ·
benign control. Plus an adaptive black-box attacker (PAIR-style), a white-box GCG suffix with a transfer test and a
tool-using agent with synthetic patient identifiers.

**Defences (8).**

| Defence | Idea |
|---|---|
| Delimiters + warning | Records inside `<trial>` tags; the system prompt says record text is untrusted data |
| Spotlighting | Every word of record text joined with `ˆ`; the model is told marked text is data (Hines et al., 2024) |
| Sandwich reminder | The task is restated after the records |
| Injection classifier | Off-the-shelf DeBERTa-v3 detector drops flagged records |
| Privilege separation | A quarantined call extracts length-capped clinical facts; a second call decides from facts only |
| Layered | Classifier + spotlighting + sandwich |
| **TrialGuard** | Rule sanitiser → registry-tuned detector → privilege separation (+ outgoing-call policy for agents) |

**Metrics.** Attack success with 95% Wilson intervals against the same outcome's base rate without an attack;
clean accuracy (best pick is a trial the patient is eligible for); exact McNemar tests against no defence on the
same patients and attacks.

**Limitations.** Small open models (4-8B), one prompt template, templated attacks plus a weak adaptive attacker,
five candidates per patient, clipped records. Results describe this benchmark, not the safety of any deployed
clinical system.
