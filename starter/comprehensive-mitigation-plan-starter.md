# Comprehensive Mitigation Plan 

# Responsible AI Risk Mitigation for a Generative AI Language Model

**System:** `distilgpt2-gender-bias-ft` (DistilGPT-2 fine-tuned on recruiting text) · **Based on:** Ethical Audit Report, 18 October 2026 · **Prepared for:** Ethics Committee review

## **1\. Overview and Objectives**

## 1.1 Purpose of This Mitigation Plan

The Ethical Audit Report found systematic gender bias in `distilgpt2-gender-bias-ft`:

* neutral roles are gendered along stereotypes (73% male for technical roles, up to 95% female for care roles);
* explicit gender cues are overridden or erased;
* women are described with 2.4× more support traits;
* hiring recommendations depend on the order of names, not merit.

It also found that the obvious quick fixes either backfire (gender-neutral instructions) or only hide the problem (pronoun replacement).

This plan turns those findings into **concrete, owned and testable safeguards**. Its objectives are to:

1. prevent the model, or any successor, from influencing real hiring or HR decisions until it passes defined bias tests;
2. reduce the measured bias at its source (the training data and the model) and contain what remains with output controls and human review;
3. make bias continuously measurable, so that deployment decisions rest on evidence;
4. meet the governance expectations for high-risk employment AI (see Appendix A).

## 1.2 Scope of Mitigation

**In scope:**
* the fine-tuned model and its fine-tuning data;
* the prompts and outputs of any application built on it (job-description drafting, candidate descriptions, comparison questions);
* the people and processes around it (reviewers, owners, monitoring, incident response).

**Out of scope:**
* retraining the base DistilGPT-2 pre-training corpus;
* protected characteristics other than gender (race, age, disability and nationality were not audited and need a separate assessment before any production use);
* non-English use;
* the design of downstream applicant-tracking systems.

**Deployment position:** the model stays **restricted to educational use**, as stated in its model card. Sections 3–6 describe what a successor model must achieve before any people-related deployment could be considered.

## **2\. Summary of Identified Risks**

| Risk Category | Description | Severity (Low / Medium / High) |
| :---- | :---- | :---- |
| Ethical | Stereotyped gendering of neutral roles (R1), overriding or erasing explicit gender (R2), stereotyped trait wording, and outputs mistaken as neutral or factual (R9) | **High** |
| Legal | Discriminatory hiring and HR outcomes if used for recruiting (R3, R4); EU AI Act high-risk obligations, anti-discrimination law, automated-decision rules; provenance of the training data (R8) | **High** |
| Privacy | Fabricated personal-data-like content such as invented social-media handles and names, and scraped-web residue (R7); personal data in prompts and logs in a real deployment | **Medium** |
| Security | Prompt manipulation (name order controls the hiring recommendation), ineffective instruction-based controls (R5), and no refusal behaviour under adversarial prompts (R6) | **High** (R5) / **Medium** (R6) |
| Environmental | Compute and energy for inference, repeated audits and retraining (R10) | **Low** |

**Full risk register (from the audit), with the safeguards that address each risk:**

| ID | Risk | Severity | Safeguards (Section 3) |
| :---- | :---- | :---- | :---- |
| R1 | Neutral roles gendered along occupational stereotypes | High | M1, M3, M4, M8 |
| R2 | Explicit gender overridden or erased | High | M1, M3, M4, M7 |
| R3 | Arbitrary, order-driven hiring recommendations with stereotyped justifications | High | M1, M5, M7 |
| R4 | Misuse in hiring, promotion or HR decisions | High | M0, M6, M7 |
| R5 | Prompt-level controls ineffective; prompts can steer outcomes | High | M3, M5, M8 |
| R6 | No refusal or safety behaviour; adversarial misuse | Medium | M3, M5 |
| R7 | Fabricated personal data and scraped-web residue | Medium | M9, M2 |
| R8 | Training-data provenance and representativeness | Medium | M2, M1 |
| R9 | Outputs mistaken as neutral; misleading aggregate metrics | Medium | M6, M8 |
| R10 | Energy and emissions from inference, audits and retraining | Low | M10 |

**What the audit already tested** (notebook "Mitigation probes" and `data_repair_and_retraining.ipynb`):

| Candidate safeguard | Result | Implication |
| :---- | :---- | :---- |
| Prompt instruction "Use gender-neutral language (they/them) only" | **Backfired:** 0% neutral outputs (from 8–26% without it); the technical role flipped to 75% female | Instructions cannot be a primary control for this model |
| Output post-processing (he/she → they, man/woman → person) | Removed 100% of gendered terms, but support traits stayed at 0.82 vs 0.34 for women- vs men-associated outputs | Surface rewriting hides the bias; it must be combined with detection and blocking |
| Data repair + retraining (CDA) | Stereotype gap **61.7 → 3.3 pp**; first-listed wins **94% → 48%**; but outputs still 85% gendered and a new male preference in hiring answers (76–82%) | Fixing the data at its source is the most effective single lever |

## **3\. Mitigation Strategies**

Each safeguard has an ID (M0–M10), the risks it addresses, an owner, a timeline and a success measure (thresholds in Section 6).

## 3.1 Bias and Fairness Mitigation

**M1: Repair the fine-tuning data and retrain (Counterfactual Data Augmentation).** *Addresses R1, R2, R3, R8.* Data control.
* For every training example, add a gender-swapped counterpart (he↔she, man↔woman, male↔female, swapped candidate names) so that every role appears equally often with each gender.
* For hiring examples, keep the question and swap the winner. Each question is then answered once for each candidate, which removes the learned "first-listed wins" pattern (85% in the original data, 50% after repair).
* Add **gender-neutral variants** (they/them, "the candidate") as a third copy, so the model learns that neutral prompts get neutral answers. Add a small set of examples where an explicit gender cue is followed faithfully (fixes R2).
* Retrain, then re-run the full bias test suite (Section 6). The prototype is `starter/data_repair_and_retraining.ipynb`; the balanced dataset is `data/gender_balanced_train.jsonl`.
* *Owner:* ML Engineering Lead · *Timeline:* 2–3 weeks · *Success:* stereotype gap ≤ 10 pp (prototype: 3.3), first-listed win rate 45–55% (prototype: 48%) and model-alone gendered rate ≤ 20% (prototype: 85%, so not yet met; requires the neutral variants).

*Prototype evidence (seeded run, `results/retraining_comparison.csv`):*

| Metric | Original | After CDA retraining |
| :---- | ----: | ----: |
| % male-framed, technical roles (neutral prompts) | 65.0 | 48.3 |
| % male-framed, support roles (neutral prompts) | 3.3 | 45.0 |
| **Stereotype gap (pp)** | **61.7** | **3.3** |
| % gendered outputs (neutral prompts) | 89.2 | 85.0 |
| **% first-listed candidate recommended** | **94.3** | **48.0** |
| % man recommended (technical / support) | 52.9 / 50.0 | 82.0 / 75.8 |

*Reading:* CDA removed the role–gender stereotype and the order shortcut, but (a) did not make the language neutral, hence the gender-neutral variants above, and (b) revealed a general male preference in comparative answers. Comparative hiring questions therefore stay blocked (M5) even after retraining.


**M2: Data governance and provenance controls.** *Addresses R8, R7.* Data control.
* Publish a datasheet for every fine-tuning dataset: source, author, purpose, composition by role type × gender, known skews, licence.
* Add an automated **pre-training bias check** that fails the pipeline when any role type exceeds a 60/40 gender split, or when the hiring winner position deviates from 50% by more than 10 pp.
* Record the base model's provenance (DistilGPT-2 / OpenWebText) and its known limitations in the model card. Store SHA-256 checksums of released weights.
* *Owner:* Data Governance Lead · *Timeline:* 4 weeks · *Success:* datasheet published; the check runs on every training job.

**M3: Model-level alignment.** *Addresses R1, R2, R5, R6.* Technical control.
* DistilGPT-2 cannot follow instructions reliably: the audit's "gender-neutral" and "fairness" instructions were ignored or backfired. Any production successor should start from an **instruction-tuned base model**.
* Add **preference tuning** (e.g. DPO) on paired examples: preferred = neutral or merit-based, rejected = stereotyped.
* Train an explicit **refusal behaviour** for discriminatory requests and for comparisons of named individuals.
* *Owner:* ML Engineering Lead · *Timeline:* 6–8 weeks · *Success:* neutral-instruction compliance ≥ 95%; red-team refusal rate 100%.

## 3.2 Prompt and Output Controls

**M4: Output bias gate (detect → regenerate → block).** *Addresses R1, R2, R9.* Technical control.
* Before any text is shown, run the audit's lexicon detector, extended with a trained bias classifier:
  1. **Gendered-language check:** if the prompt contains no gender cue and the output contains gendered terms, regenerate up to 2 times, then neutralise and flag the output for review.
  2. **Cue-fidelity check:** if the prompt states a gender and the output uses the other one, block the output (this catches the "male assistant → woman" and "She → he" erasures).
  3. **Trait-balance check:** flag outputs whose support-trait vs leadership-trait counts fall outside the range observed for gender-swapped control texts.
  4. **Counterfactual consistency check (sampled):** for 5% of requests, generate the gender-swapped prompt too and compare lexicon signals. Large differences are escalated to monitoring (M8).
* Pronoun neutralisation is used **only as a last step after detection**. The audit showed it hides the bias without removing it.
* *Owner:* ML Platform Engineer · *Timeline:* 3–4 weeks · *Success:* ≤ 5% gendered outputs for neutral prompts after the gate; 0 cue-fidelity violations in the test suite.

**M5: Input controls and use-case restriction.** *Addresses R3, R5, R6.* Technical control.
* **Block comparative hiring questions about named people** ("Who is a better fit, X or Y?"). These are refused with a fixed message pointing to the structured, human-run selection process. The audit showed these answers are driven by name order (first-listed wins 78–84%) and not by merit.
* **Structured, blind templates:** the application builds prompts from approved fields (role title, responsibilities, required skills) and strips names, pronouns and other demographic attributes from free text before generation.
* **Adversarial input filter:** detect and refuse requests for exclusionary content ("only men", "young women", "need not apply") and attempts to override instructions.
* Maintain a **red-team suite** (the audit's red-team prompts plus new cases each quarter) as a release test.
* *Owner:* Application Security Lead · *Timeline:* 2–3 weeks · *Success:* 100% of comparative and exclusionary requests refused in tests; red-team suite passes before each release.

## 3.3 Explainability and Monitoring

**M8: Standing bias test suite, monitoring and incident response.** *Addresses R1, R5, R9.* Process control.
* **Release gate:** the audit notebook becomes an automated suite with fixed seeds. Every model or prompt change must pass the Section 6 thresholds. The suite must report rates over repeated samples, order-controlled hiring results and lexicon trait shares, **not** single Jaccard scores or aggregate selection parity, which the audit showed can hide bias.
* **Production monitoring:** a weekly dashboard of gendered-output rate for neutral prompts, cue-fidelity violations, bias-gate trigger rate, reviewer overrides and user reports. An alert fires if the gendered-output rate exceeds 5% in any week.
* **Incident response:** severity tiers. SEV-1 is a biased output that reached a candidate or influenced a decision; SEV-2 is a repeated gate failure; SEV-3 is an isolated flag. SEV-1 triggers immediate feature suspension, notification of Legal and HR, root-cause analysis and a regression test.
* **Explainability for reviewers:** each output shown to a reviewer carries its lexicon signals (gendered terms, trait counts) so the reviewer can see *why* it was flagged.
* *Owner:* Responsible AI Lead · *Timeline:* 4 weeks to set up, then ongoing · *Success:* suite runs on every release; dashboard live; quarterly re-audit completed.

**M9: Privacy safeguards.** *Addresses R7.* Technical and process control.
* Add an output filter for personal-data patterns (emails, phone numbers, @handles, URLs, invented "contact" details) using the redaction approach from the data-leak module, extended to social handles.
* Minimise logs: store prompts and outputs with candidate names pseudonymised, use role-based access and a 90-day retention period.
* *Owner:* Privacy Officer · *Timeline:* 3 weeks · *Success:* 0 handles or emails in sampled outputs; log retention enforced.

## 3.4 Human Oversight and Governance

**M0: Use restriction and deployment gate.** *Addresses R4.* Governance control.
* The model is classified as **"educational only – not for people decisions"** in the model registry. Access is limited to training environments, and its model card warning is shown wherever the model is used.
* Any people-related deployment requires approval by the Ethics Committee / Responsible AI Review Board after the Section 6 criteria are met, a Data Protection Impact Assessment and a legal review against the EU AI Act high-risk requirements.
* *Owner:* Head of Responsible AI · *Timeline:* immediate · *Success:* no production endpoint exists without a signed approval.

**M6: Transparency and user guidance.** *Addresses R4, R9.* Policy and communication control.
* Label all generated text as AI-generated and as a draft.
* Update the model card with this audit's quantitative findings and the explicit statement that aggregate parity does not mean fairness.
* Train recruiters and reviewers to recognise stereotyped wording, using examples from the audit.
* *Owner:* Product Manager with HR Business Partner · *Timeline:* 3 weeks · *Success:* 100% of reviewers trained; labels present in 100% of outputs.

**M7: Human-in-the-loop review.** *Addresses R2, R3, R4.* Process control.
* No generated text about roles or people is published or used without review by a named human: the hiring manager or recruiter, following the human-review workflow pattern.
* The reviewer can **approve, edit, reject or escalate**. Escalation goes to an HR/DEI specialist for anything involving candidate comparisons, protected characteristics or flagged outputs.
* Every decision is logged with reviewer, time, action and reason. Rejections feed back into the bias test suite and the training data.
* The model **never** ranks or selects candidates.
* *Owner:* Head of Talent Acquisition · *Timeline:* 4 weeks · *Success:* 100% of outputs reviewed before use; reviewer override rate tracked; no automated decisions.

**M10: Environmental efficiency.** *Addresses R10.* Process control.
* Cache evaluation outputs, run the full bias suite per release (not continuously), and track the compute and energy of training and audit runs in the release notes.
* Prefer small, efficient models where they meet the fairness criteria.
* *Owner:* ML Platform Engineer · *Timeline:* ongoing · *Success:* energy per release recorded.

## **4\. Implementation Plan**

## 4.1 Technical Implementation

| Phase | Weeks | Work | Safeguards |
| :---- | :---- | :---- | :---- |
| 0. Contain | Week 0 | Registry flag "educational only"; access restriction; model card warning | M0 |
| 1. Measure | Weeks 1–4 | Convert the audit notebook into an automated, seeded test suite with thresholds; add a CI job; build the monitoring dashboard | M8, M10 |
| 2. Repair | Weeks 1–3 | CDA + neutral variants + cue-fidelity examples; dataset checks; datasheet; retrain and re-test | M1, M2 |
| 3. Guard | Weeks 2–5 | Input filters (block comparative and exclusionary requests, blind templates); output bias gate; PII filter | M4, M5, M9 |
| 4. Align | Weeks 5–12 | If a production successor is pursued: instruction-tuned base model + preference tuning + refusal training | M3 |
| 5. Validate | Week 12+ | Full re-audit against Section 6; red-team exercise; legal and DPIA review | M8, M0 |

The architecture places controls **around** the model rather than relying on the model itself: request → input filter and blind template → model → output bias gate and PII filter → human reviewer → use. Every step writes to an audit log.

## 4.2 Operational Implementation

* **Governance body:** a Responsible AI Review Board (Responsible AI Lead, ML Engineering, Legal/Compliance, HR/DEI, Privacy Officer) owns the risk register and approves deployments. It meets monthly during implementation and quarterly afterwards.
* **Accountability:** every safeguard has a named owner (Section 3), and progress is tracked in the risk register with status, evidence and due date.
* **Policies:**
  * an acceptable-use policy that bans using generative models to rank, select or reject candidates;
  * a job-ad language standard (gender-neutral wording, inclusive-language checklist);
  * a procedure for candidates and employees to report biased content.
* **Escalation:** owners report a missed threshold or SEV-1 incident to the Board within 24 hours. The Board can suspend the feature.
* **Training:** onboarding for reviewers (M6/M7), plus an annual refresher based on new audit findings.

## **5\. Residual Risk Assessment**

| Risk | Remaining Risk Level | Rationale |
| :---- | :---- | :---- |
| Gender bias in neutral roles | **Medium** | CDA retraining and the output gate reduce the stereotype gap and the gendered-output rate (see the M1 retraining evidence). But lexicon detection misses subtle framing, and pre-training associations remain in the base model. Human review and monitoring are still required. |
| Misuse in hiring decisions | **Low** (with M0, M5, M7 enforced) | Comparative and ranking requests are blocked, the model is restricted from people decisions, and every output is reviewed by a named human. Residual risk comes from people bypassing policy (e.g. copying text into other tools), which is handled through training, logging and audits. |
| Prompt manipulation | **Medium** | Blocking comparative questions removes the order-driven decision path, and input filters catch known adversarial phrasing. New jailbreak styles will appear, and this model cannot reliably follow instructions, so quarterly red-teaming remains necessary. |
| Erasure of explicit gender (R2) | Medium → Low | The cue-fidelity check blocks contradicting outputs. It depends on the lexicon covering names and pronouns, and misses implicit cues. |
| Fabricated personal data (R7) | Low | The PII/handle filter and human review catch most cases, but entirely fabricated names cannot always be detected. |
| Other protected characteristics | **Unknown** | Not tested in this audit. Must be assessed before any production use. |
| Environmental (R10) | Low | Small model; compute is tracked. |

## **6\. Validation and Success Criteria**

A successor model and application may only be proposed for deployment when **all** of the following hold on the automated bias suite. The suite uses fixed seeds, at least 50 samples per case, the audit prompts plus held-out prompts, and greedy plus sampled decoding.

| Metric (from the audit notebook) | Audit baseline | Target |
| :---- | :---- | :---- |
| Gendered outputs for neutral prompts (after the output gate) | 74–92% | ≤ 5% |
| Gendered outputs for neutral prompts (model alone, before the gate) | 74–92% | ≤ 20% |
| Stereotype gap: % male framing for technical roles − % for support roles | 57 pp | ≤ 10 pp |
| Explicit-cue fidelity (output matches the stated gender) | 0–100% (male support role: 0%) | ≥ 95% for every role |
| Compliance with a gender-neutral instruction | 0% | ≥ 95% |
| Leadership-share difference between female- and male-framed outputs | 0.12 (0.30 vs 0.42) | ≤ 0.05 |
| Support-trait ratio, female vs male framing | 2.4× | ≤ 1.2× |
| Comparative hiring questions about named people | answered; first-listed wins 78–84% | 100% refused (in a test mode without refusal: first-listed 45–55% and man selected 45–55% in **both** orders) |
| Red-team refusal of discriminatory requests | 0% | 100% |
| Personal-data artifacts (handles, emails) in sampled outputs | present | 0 |
| Human review coverage before use | n/a | 100%, with logged decisions |

**Validation process:**
1. ML Engineering runs the suite.
2. The Responsible AI Lead independently reproduces it.
3. Red-teamers who did not build the safeguards test them.
4. The Board signs off on the evidence.

After deployment, the same metrics are monitored weekly (M8) and fully re-audited every quarter, or after any model, prompt or data change.

## **7\. Limitations of the Mitigation Plan**

* **Lexicon detectors are approximate.** They miss synonyms, implicit framing and context, and they produce false positives (e.g. "he" inside a quotation). The bias gate reduces risk but cannot guarantee neutral text, which is why human review stays mandatory.
* **CDA balances the binary gender split but does not teach fairness reasoning.** Retraining a biased checkpoint can leave residual associations from pre-training, and gender-swapping can create unnatural examples (e.g. "she" in a male-coded sentence).
* **Small base model.** DistilGPT-2 is incoherent and does not follow instructions, which limits what model-level fixes can achieve. A production use case would likely need a different, larger base model, with its own audit and higher compute cost.
* **Only gender was audited,** in English, for one domain. Intersectional and other protected-attribute risks remain unknown.
* **Human reviewers have biases too.** Training and decision logging reduce, but do not remove, automation bias and reviewer bias.
* **Owners and timelines** are planning assumptions for a course project and would need confirmation by the actual organisation.

## **8\. Conclusion**

Summary:
* **Risks:** the audit found High ethical, legal and security risks for any people-related use of `distilgpt2-gender-bias-ft`, Medium privacy risk and Low environmental risk.
* **What doesn't work:** the cheap fixes (prompt instructions, pronoun replacement) do not work on their own.
* **The plan:** contain the model immediately (M0); repair the training data and retrain (M1, M2); guard every request and response with input filters, an output bias gate and privacy filters (M4, M5, M9); keep a named human in the loop with no automated decisions (M7); measure continuously against fixed thresholds (M8).
* **Decision requested:** approve the containment measures and the implementation roadmap, and **reject deployment of the current model** for any recruiting or HR use. A successor may return to the Committee only with evidence that it meets every Section 6 criterion.

## Appendix A. Regulatory Mapping (EU AI Act and NIST AI RMF)

*This mapping shows how the controls in this plan line up with two widely used frameworks. It is a planning aid, not legal advice; Legal and Compliance must confirm obligations for any real deployment.*

### A.1 EU AI Act (Regulation (EU) 2024/1689)

AI systems used in **employment** (recruitment, filtering applications, evaluating candidates, promotion decisions) are listed as **high-risk** in Annex III, point 4. If this model, or a model like it, were used in hiring in the EU, the following obligations would apply. The table maps each one to the safeguards in this plan.

| EU AI Act requirement | What it means here | Safeguards in this plan |
| :---- | :---- | :---- |
| Art. 9: Risk management system | A documented, continuous process to identify, evaluate and reduce risks | §2 risk register, §5 residual risk, §6 release gates, quarterly reviews (§4.2) |
| Art. 10: Data and data governance | Training data must be relevant, representative and examined for bias | M1 data repair (CDA), M2 dataset documentation and bias checks before training |
| Art. 11 and Annex IV: Technical documentation | Documented design, data, testing and limitations | Updated model card, audit report, evaluation notebooks and results tables kept in version control |
| Art. 12: Record-keeping (logging) | Automatic logs of system use to support traceability | M8 audit logging of prompts, outputs, guardrail decisions and reviewer actions |
| Art. 13: Transparency to deployers | Clear instructions, capabilities and limitations | Model card "out-of-scope" warnings, user-facing disclaimer (M6) |
| Art. 14: Human oversight | Humans can understand, override and stop the system | M7 human-in-the-loop review, no automated hiring decisions, kill switch |
| Art. 15: Accuracy, robustness, cybersecurity | Consistent behaviour, resilient to manipulation | M5 adversarial/red-team test suite, prompt-injection filters, robustness tests across prompt variants |
| Art. 5: Prohibited practices / Art. 50 transparency | Users must know they interact with AI-generated content | AI-generated content labels (M6) |

### A.2 NIST AI Risk Management Framework (AI RMF 1.0)

| NIST function | Purpose | How this plan covers it |
| :---- | :---- | :---- |
| **GOVERN** | Culture, roles, accountability, policies | Named owners for every mitigation, Responsible AI Review Board, acceptable-use policy, deployment gate (§4.2) |
| **MAP** | Understand context, intended use and impacts | Intended-use and out-of-scope definitions, stakeholder impact analysis (candidates, recruiters, the organisation), risk register (§2) |
| **MEASURE** | Quantify and track risks with suitable metrics | Bias test suite with fixed thresholds (§6): neutral-prompt gendering rate, stereotype gap, hiring selection parity, counterfactual shift, red-team pass rate |
| **MANAGE** | Prioritise, respond to and monitor risks | Mitigations M1–M10, residual-risk acceptance (§5), monitoring dashboards and incident response (§3.3) |

NIST's characteristics of trustworthy AI addressed most directly: **fair, with harmful bias managed** (M1–M4); **accountable and transparent** (M6–M8); **valid and reliable** (§6 metrics); **secure and resilient** (M5); **privacy-enhanced** (M9).

