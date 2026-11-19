# Action Plan: Mechanistic Interpretability for Prefix-Level Hallucination Detection

## 1. Background and Positioning

**Base paper:** Harary et al., *PrefixNLI: Detecting Factual Inconsistencies as Soon as They Arise* (arXiv 2511.01359). The paper introduces MiniTruePrefixes, a LLaMA-3.2-1B NLI model fine-tuned to judge whether a text prefix is entailed by its source. It uses the output probability of the "1" token with a fixed threshold (τ = 0.5) and applies a log-odds penalty during controlled decoding.

**Assessment:** The base paper is **not mechanistic interpretability**. It is a black-box, output-level method. Training updates only the final layer, and the verdict is read from output probabilities. No internal representations, circuits, or causal interventions are analyzed.

**Our contribution has two parts:**

1. **Statistical:** A calibrated, prefix-length-aware decision threshold to replace the fixed τ = 0.5 (non-MI, calibration).
2. **Mechanistic:** Explain where and how the entailment decision forms inside the model, and validate it causally (MI).

**Research question:** Where in MiniTruePrefixes does the unfaithfulness signal for a prefix emerge, is it causally used for the verdict, and can a mechanistically grounded score detect hallucinations earlier and more reliably than the output probability?

## 2. Hypotheses

- **H1 (Localization):** The unfaithfulness signal is linearly decodable from intermediate layers before the final layer, and becomes decodable earlier for longer prefixes.
- **H2 (Causality):** A small set of attention heads or MLP directions causally drives the verdict. Ablating them changes predictions on faithful and unfaithful prefixes.
- **H3 (Failure mechanism):** False positives (33% of FP in the base paper) are driven by lexical-overlap components, and can be identified and reduced by targeted intervention.
- **H4 (Early detection):** An internal-probe score beats output probability for prefixes in the 0–32% length bin, where the base paper reports F1 of only 27.0.
- **H5 (Calibration):** A prefix-length-aware threshold controls the false-negative rate at a target level better than τ = 0.5.

## 3. Experimental Plan

### Phase 1: Reproduction (Weeks 1–2)

- Obtain code and models from https://github.com/sapirharary/PrefixNLI.
- Reproduce the intrinsic evaluation on RAGTruthPrefixes and SummEditsPrefixes. Target F1 (unfaithful class): SummEdits 78.1, RAGTruth 47.6.
- Reproduce the baseline comparison against MiniTrue and MiniCheck.
- Reproduce the per-prefix-length results (Figure 4).
- **Deliverable:** Reproduction report with confidence intervals matching the paper.

### Phase 2: Layer-wise Probing (Weeks 3–4)

- Extract residual-stream activations at the verdict position for every layer of MiniTruePrefixes.
- Train linear probes (logistic regression with L2 regularization) per layer to predict gold entailment labels.
- Stratify by prefix-length bin (0–32%, 32–64%, 64–100%, 100%).
- Controls: random-direction probes, length-only probes, and probes trained on shuffled labels.
- **Deliverable:** Layer-by-prefix-length probe accuracy heatmap, plus a figure showing where the signal first becomes decodable.

### Phase 3: Causal Localization (Weeks 5–6)

- Activation patching: for FP and FN examples, patch activations from a correct run into an incorrect run, one component at a time (attention heads, MLP outputs).
- Head and MLP ablations: zero or mean-ablate candidate components and measure verdict flips.
- Use the probe direction from Phase 2 to test whether intervening along it changes the verdict.
- **Deliverable:** Ranked list of causally important components with effect sizes and error bars.

### Phase 4: Failure Analysis (Week 7)

- Take the base paper's error categories (surface-level heuristic bias, generic prediction error, missed inference) and check which components drive each.
- Test whether lexical-overlap heads are necessary for the surface-heuristic false positives, by ablating them and checking whether those errors disappear.
- **Deliverable:** Mapping from error type to internal mechanism.

### Phase 5: Detection and Calibration (Weeks 8–9)

- Build the internal-probe detector and compare it to output-probability detection (the base paper's method) on AUROC and F1, separately for each prefix-length bin.
- Statistical cutoff: calibrate per prefix length using conformal prediction or a target false-negative rate. Compare against the fixed τ = 0.5 and against a global calibrated threshold.
- Report calibration error (ECE) for both the output score and the probe score.
- **Deliverable:** Detector comparison table and calibration plots.

### Phase 6: Causal Steering in Decoding (Week 10)

- Apply the identified direction or ablation during controlled decoding on CNN/DM and XSum, using the base paper's setup (LLaMA-3.2-1B/3B/8B generators, MiniCheck and GPT-4.1 faithfulness metrics, ROUGE-L and MAUVE for quality).
- Compare against the base paper's Prefix method and Vanilla decoding.
- **Deliverable:** Faithfulness vs. quality vs. latency table.

### Phase 7: Robustness and Write-up (Weeks 11–12)

- Robustness: logically constructed contradictions (e.g., the ContraTales benchmark, arXiv 2507.23221), cross-dataset transfer, and a second NLI model family if available.
- Write the paper. Target venue: interpretability or safety workshop first, then a main track.

## 4. Experimental Controls (Required)

- Length-matched and position-matched baselines, so probes do not track length.
- Random-direction and random-head ablations.
- Three or more seeds for probe training; report confidence intervals (bootstrap, 1,000 resamples, as in the base paper).
- Held-out split for all threshold calibration. Never calibrate and evaluate on the same data.
- Second NLI judge (e.g., MiniCheck) to check label noise.

## 5. Resources and Compute

- **Models:** MiniTruePrefixes (1B), LLaMA-3.2-1B/3B/8B-Instruct generators, OLMo 1B/7B for generalization.
- **Datasets:** RAGTruthPrefixes, SummEditsPrefixes, CNN/DM and XSum (decoding), ContraTales (robustness).
- **Compute:** Single A100 80GB is sufficient for Phases 1–5. Phase 6 needs about 45 GPU-hours, per the base paper.
- **Tooling:** TransformerLens or nnsight for activation access; vLLM for decoding; scikit-learn for probes.

## 6. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Probes track length or position, not faithfulness | Length-matched controls, random-direction baselines |
| Causal effects are small or distributed across many components | Report negative results; use path patching if single-component effects are weak |
| Base model is too small for clear mechanisms | Replicate on the 3B and 8B generators' representations if a probe is trained on them |
| Label noise in RAGTruth and SummEdits | Second judge, human spot-check of 100 disagreements |
| Similar work appears during the project | Differentiate on prefix-level causality and calibrated early-stopping |

## 7. Deliverables Summary

1. Reproduction report (Week 2)
2. Layer-wise probe results (Week 4)
3. Causal component map (Week 6)
4. Failure-mechanism analysis (Week 7)
5. Detector and calibration comparison (Week 9)
6. Steering results in controlled decoding (Week 10)
7. Draft paper and code release (Week 12)

## 8. Immediate Next Steps

1. Clone https://github.com/sapirharary/PrefixNLI and run the evaluation script.
2. Confirm access to RAGTruth and SummEdits, and check their licenses.
3. Set up activation extraction for MiniTruePrefixes and verify it on five examples.
4. Decide the threshold-calibration method (conformal vs. target FNR) before Phase 5.
5. Share your paper's specific contributions so this plan can be narrowed to what is new in your work.
