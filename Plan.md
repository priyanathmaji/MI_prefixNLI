# Research Plan: Mechanistic Analysis of Prefix-Level Hallucination Detection

**Prepared for:** Research Manager
**Prepared by:** Priyanath Maji / pmaji3@gatech.edu
**Date:** 11/18/2026
**Status:** Draft for friendly review (v2: adds judge-free SAE extension)

---

## 1. Summary

Large language models sometimes generate statements that are not supported by their source material. Current detectors check these statements only after a full answer is written. A recent method, PrefixNLI (Harary et al., 2025), checks each partial answer as it is generated, which allows errors to be caught earlier. However, PrefixNLI relies on a separate external judge model, which adds cost and is itself a black box.

This project extends PrefixNLI in three ways:

1. **Better decision thresholds.** The existing method uses a fixed cutoff (0.5) for every situation. We will calibrate the cutoff statistically, so that the rate of missed errors is controlled and known in advance.
2. **Understanding the model's internals.** We will examine the detector's internal workings to learn where and how it recognizes unsupported content, and test whether those internal signals are the actual cause of its decisions.
3. **Judge-free detection from the generator itself (new in v2).** Instead of relying on an external NLI judge, we will test whether the text generator's own internal activations, decomposed into interpretable features with sparse autoencoders (SAEs), already carry a signal that flags unsupported content as it is produced. If they do, hallucinations can be detected with no second model and no extra judge calls.

The result will be a detector that is more reliable, cheaper to run, and explainable, which matters for deploying AI systems safely.

## 2. Goals

- Determine whether the detector's internal signals identify unsupported content earlier than its output does.
- Replace the fixed cutoff with a calibrated one, and measure the improvement.
- Identify which internal components cause the detector's errors, and whether they can be corrected.
- **New:** Determine whether sparse, interpretable SAE features in the generator flag unsupported content during generation, without an external judge.
- **New:** Test whether those features causally influence whether the generator produces unsupported content.
- Produce a paper suitable for an AI safety or interpretability workshop, followed by a main-track submission.

## 3. Scope

**In scope:** Analysis of an open 1-billion-parameter detector model; analysis of an open generator model with an existing pretrained SAE; evaluation on two existing public benchmarks; use of the generator to produce news summaries and retrieval-augmented answers.

**Out of scope:** Training new large models; training new SAEs unless no suitable pretrained SAE exists (see Risks); building a production system; evaluation on non-English text or domains other than news and retrieval-augmented answers.

## 4. Timeline

| Weeks | Milestone | Outcome |
|---|---|---|
| 1–2 | Reproduce published results; confirm generator model and pretrained SAE availability | Confirmed baseline numbers; chosen generator |
| 3–4 | Locate where the detector's internal signals form | Map of signal strength by model depth and answer length |
| 5–6 | Identify components that cause decisions | Ranked list of influential components |
| 7 | Analyze error types | Explanation of main failure modes |
| 8–9 | Compare detection methods and calibrate thresholds | Head-to-head results against the published method |
| 10–11 | **Extension A:** Extract SAE features from the generator on labeled prefixes; discover candidate features | Candidate feature set; feature interpretations |
| 12 | **Extension B:** Evaluate judge-free SAE detector against the NLI judge and baselines | Detection comparison, including cost per prefix |
| 13 | **Extension C:** Test causal effect of features on generation; combine with generation experiment | Faithfulness, quality, and speed results |
| 14–15 | Robustness checks across datasets and controls | Robustness results |
| 16 | Write-up and code release | Draft paper and code release |

**Total duration:** 16 weeks (previously 12; the extension adds 4 weeks).

## 5. Points for Discussion

1. **Extension:** Feedback on the judge-free SAE extension and the revised 16-week timeline.
2. **Threshold method:** Input on either a conformal prediction approach (guarantees a target error rate under standard assumptions) or a simpler target-based method. Recommendation: conformal prediction, as it provides a stronger guarantee.
3. **Generator model:** Thoughts on the generator to use. Recommendation: choose a model for which a pretrained SAE already exists, to avoid training one. This will be verified in Weeks 1–2.
4. **Target venue:** Thoughts on whether the first target is a workshop (faster, lower bar) or a main conference track (slower, higher bar). Recommendation: workshop first, then main track.
5. **Compute:** Estimated at roughly 100–140 GPU-hours on a single A100-class GPU (estimate; see Section 6), plus storage for sparse SAE features. Storage will be sized in a Week 2 pilot.
6. **Ownership:** Thoughts on who will own the code release and data licensing checks.

## 6. Resources

- **Compute:** One A100 80GB GPU, estimated at 100–140 hours in total. The largest shares are generation experiments (about 45 hours) and SAE feature extraction over labeled prefixes (estimated 20–30 hours). These are estimates and will be refined after the Week 2 pilot.
- **Storage:** Sparse SAE features are stored instead of raw activations to keep storage manageable. The final size will be determined in the Week 2 pilot.
- **Data:** Two public benchmarks (RAGTruth and SummEdits) and two summarization datasets (CNN/DM and XSum). All are publicly available; licenses will be checked before use. Benchmark annotations are used as labels for training and evaluation only; no external judge is needed at detection time.
- **Models:** One open detector model (1B); one open generator model with a matching pretrained SAE (to be confirmed).
- **Software:** Open-source tools for model analysis, SAE encoding, and inference.
- **People:** Individual

## 7. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Observed signals reflect answer length rather than accuracy | Misleading conclusions | Include controls that remove length effects |
| Causal effects are weak or spread across many components | Less clear findings | Report negative results; use broader analysis methods |
| Labels in the benchmarks are noisy | Lower measured accuracy | Use a second judge and manually review a sample of disagreements |
| A similar study is published during the project | Reduced novelty | Focus on the combination of calibration, judge-free detection, and causal analysis |
| Detector is too small to show clear mechanisms | Limited findings | Repeat key experiments on a larger model if the results justify it |
| **No suitable pretrained SAE for the chosen generator** | Adds several weeks and compute | Choose a generator with an available SAE in Weeks 1–2; if none exists, scope down to one layer and budget for SAE training |
| **SAE features track topic or style rather than faithfulness** | Features look meaningful but do not detect errors | Use contrastive prefix pairs (faithful vs. unfaithful on the same source), topic-matched controls, and random-feature baselines |
| **Feature interpretation is subjective** | Weak or disputed labels for features | Pre-register labeling criteria; two annotators; report agreement |
| **Judge-free signal is weaker than the NLI judge** | Extension does not improve detection | Report the gap and its cause; the NLI judge remains available as a fallback, so the project still produces results |
| **SAE reconstruction error hides relevant information** | Missed signal | Compare SAE-based detection against a linear probe on raw activations as a control |

## 8. Success Criteria

- The calibrated detector meets its target error rate on held-out data.
- At least one internal component is shown to causally influence the detector's decisions, with a measured effect.
- Detection of early errors (the first third of an answer) improves over the published method.
- **New:** The judge-free SAE detector matches or exceeds the NLI-based detector on early prefixes, or the gap and its cause are clearly explained.
- **New:** At least one SAE feature set causally changes faithfulness in generation, with a measured effect beyond random-feature controls.
- Results are reproducible with released code.

## 9. Deliverables

1. Reproduction report and generator/SAE confirmation (Week 2)
2. Internal signal analysis of the detector (Week 4)
3. Causal component analysis of the detector (Week 6)
4. Comparison and calibration results (Week 9)
5. SAE feature candidates and interpretations (Week 11)
6. Judge-free detector evaluation (Week 12)
7. Causal feature intervention and generation results (Week 13)
8. Draft paper and code release (Week 16)

## 10. Next Steps

1. Friendly review of this plan and the points in Section 5.
2. Access the published detector code and benchmark datasets.
3. Identify a generator model with a matching pretrained SAE, and run a sizing pilot for storage and compute.
4. Run the first reproduction check within two weeks of starting the project.
