# Network Intrusion Detection with Machine Learning
### CIC-IDS2017 · Random Forest vs Neural Network · Cross-Day Generalization Analysis

**Author:** Shanzay Jamil · [GitHub](https://github.com/shay-coder) · 

A machine-learning-based Intrusion Detection System (IDS) built on **CIC-IDS2017**, the
benchmark dataset from the Canadian Institute for Cybersecurity (UNB). The project goes beyond
a single accuracy number: it deliberately evaluates the model twice — once the *easy* way
(random split) and once the *honest* way (train on some days, test on unseen days and unseen
attack families) — and analyses the gap between them.

---

## 📁 Repository structure

```
├── Network_Intrusion_Detection.ipynb   # Main pipeline: EDA → cleaning → RF & MLP → evaluation
├── Cross_Day_Generalization.ipynb      # The harder test: unseen days & unseen attack types
├── data/                               # Dataset CSVs go here (not committed — see data/README.md)
├── requirements.txt
└── README.md
```

## 🚀 Quick start

```bash
git clone https://github.com/shay-coder/network-intrusion-detection.git
cd network-intrusion-detection
pip install -r requirements.txt
# Download CIC-IDS2017 CSVs into data/ (instructions in data/README.md)
jupyter notebook Network_Intrusion_Detection.ipynb
```

## 🧪 What's in the main notebook

| Step | What happens | Why |
|------|--------------|-----|
| EDA | Class-balance analysis | Imbalanced data makes plain accuracy misleading |
| Cleaning | inf→NaN→drop, **de-duplication**, identifier columns removed | Duplicates across train/test = leakage; IPs/timestamps = shortcut features |
| Split | 80/20, **stratified** | Preserves attack/benign ratio in both sets |
| Models | Random Forest (raw features) · MLP 64→32 (standardized, early stopping) | Tree ensembles vs neural nets on tabular data |
| Evaluation | Accuracy + **Attack Recall** + F1, confusion matrix, feature importance | For an IDS, missed attacks (recall) matter more than accuracy |

### Results — within-day evaluation (random 80/20 split)

| Model | Accuracy | Attack Recall | F1 |
|-------|----------|---------------|------|
| Random Forest | **99.90%** | ~0.999 | 0.9983 |
| Neural Network (MLP) | 99.42% | ~0.99 | 0.9900 |

Random Forest wins — consistent with the well-documented pattern that tree ensembles
outperform standard neural networks on engineered tabular features. Top predictive features
are flow **timing (IAT)**, **duration**, and **packet-length statistics**.

### Results — cross-day evaluation (`Cross_Day_Generalization.ipynb`)

| Setup | Accuracy | Attack Recall | F1 |
|-------|----------|---------------|-----|
| Within-day baseline | 99.90% | ~0.999 | 0.9983 |
| Train Tue+Wed → Test Friday (DDoS + unseen PortScan) | _run to fill_ | _run to fill_ | _run to fill_ |
| Train all other days → Test Thursday (unseen Web Attacks) | _run to fill_ | _run to fill_ | _run to fill_ |

**The point of this table:** the drop between row 1 and rows 2–3 is the *generalization gap* —
the honest measure of how an IDS would behave against traffic it has never seen. The per-attack
breakdown in the notebook shows *which* attack families transfer (volumetric: DDoS, PortScan)
and which don't (content-based: XSS, SQLi).

## 📌 Key findings

1. **Within-day detection of volumetric attacks is essentially solved** by a plain Random
   Forest on flow features — DDoS and PortScan distort traffic *shape* so strongly that they
   are nearly separable out of the box.
2. **Accuracy is the wrong headline metric.** With ~71/29 class balance, we report attack
   recall and F1 throughout; for an IDS a false negative (missed intrusion) is far costlier
   than a false positive (wasted analyst time).
3. **Honest evaluation changes the story.** De-duplication, stratification, and especially
   cross-day splits reveal the real difficulty hidden by a naive random split.
4. **Flow features are structurally blind to content-based attacks.** SQLi/XSS live in the
   HTTP payload; flow statistics describe only traffic shape. This motivates payload-aware
   and behavioural-graph approaches rather than more data or bigger models.

## ⚠️ Assumptions & limitations (read before citing numbers)

- **Binary framing.** All attack types are collapsed to ATTACK=1. Attack-type identification
  (multi-class) is future work.
- **Flow-level features only.** No payload/content inspection; per-packet raw bytes are not used.
- **Subsampling.** 50k rows sampled per CSV for tractability (random sample, seeded — not the
  biased "first N rows"). Rare classes (Heartbleed, Infiltration) may be under-represented;
  per-class recalls on small classes are noisy.
- **Single lab environment.** Both notebooks train *and* test inside CIC-IDS2017 — one network
  topology, one week, simulated user behaviour. Even the cross-day experiment does not capture
  true deployment shift (different network/era). Cross-*dataset* evaluation (e.g., test on
  CSE-CIC-IDS2018) is the natural next step.
- **Known dataset caveats.** CIC-IDS2017 has documented labelling/flow-construction issues
  (e.g., Engelen et al., 2021; Lanvin et al., 2023); results on the corrected releases
  (WTMC/“improved” versions) can differ.
- **No hyperparameter tuning.** Defaults + fixed seeds, deliberately — the comparisons isolate
  data-handling effects, not tuning effort.

## 🗺️ Roadmap

- [ ] Multi-class attack-type classification with per-class confusion matrix
- [ ] Class-imbalance treatments (class weights, SMOTE) targeting Web-Attack recall
- [ ] 5-fold stratified cross-validation (mean ± std instead of single-split numbers)
- [ ] Gradient-boosted trees (XGBoost / LightGBM) benchmark
- [ ] Feature selection: retrain on top-15 features for real-time feasibility
- [ ] Cross-dataset test on CSE-CIC-IDS2018
- [ ] Minimal live demo: CICFlowMeter output → trained model → alerts

## 📚 Dataset & citation

CIC-IDS2017 — Canadian Institute for Cybersecurity, University of New Brunswick.
Download instructions in [`data/README.md`](data/README.md). If you use the dataset, cite:

> Sharafaldin, I., Habibi Lashkari, A., & Ghorbani, A. A. (2018). *Toward Generating a New
> Intrusion Detection Dataset and Intrusion Traffic Characterization.* ICISSP 2018.

## License

MIT — see [LICENSE](LICENSE).
