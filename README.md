# Network Intrusion Detection with Machine Learning
### CIC-IDS2017 · Random Forest vs Neural Network · Cross-Day Generalization Analysis

**Author:** Shanzay Jamil · [GitHub](https://github.com/shay-coder) · 

A machine-learning-based Intrusion Detection System (IDS) built on **CIC-IDS2017**, the benchmark
dataset from the Canadian Institute for Cybersecurity (UNB). The project deliberately evaluates
the same model twice, once the *easy* way (random split) and once the *honest* way (train on
some days, test on unseen days and unseen attack families) and analyses the gap between them.

**Result** attack recall goes

> **0.999** (random split) → **0.345** (unseen day) → **0.003** (unseen web attacks)

A model that looks essentially perfect under standard evaluation misses 2 of every 3 attacks the
first time it faces a new day, and is blind to content-based attacks, while still reporting
98.66% *accuracy*. Measuring and explaining that gap is the point of this repository.

---

## Repository structure

```
├── Network_Intrusion_Detection.ipynb   # Main pipeline: EDA → cleaning → RF & MLP → evaluation
├── Cross_Day_Generalization.ipynb      # The honest test: unseen days & unseen attack families
├── figures/                            # Exported charts (see figures/README.md)
├── data/                               # Dataset CSVs go here (not committed — see data/README.md)
├── requirements.txt
└── README.md
```

## Steps

```bash
git clone https://github.com/shay-coder/network-intrusion-detection.git
cd network-intrusion-detection
pip install -r requirements.txt
# Download CIC-IDS2017 CSVs into data/ (instructions in data/README.md)
jupyter notebook Network_Intrusion_Detection.ipynb
```

Runs on Google Colab too: mount Drive and set `DATA_DIR = '/content/drive/MyDrive/data/'`
(a commented line is provided in the notebooks).

##  Part 1 — Main notebook (within-day evaluation)

| Step | What happens | Why |
|------|--------------|-----|
| EDA | Class-balance analysis | Imbalanced data makes plain accuracy misleading |
| Cleaning | inf→NaN→drop, **de-duplication**, identifier columns removed | Duplicates across train/test = leakage; IPs/timestamps = shortcut features |
| Split | 80/20, **stratified** | Preserves attack/benign ratio in both sets |
| Models | Random Forest (raw features) · MLP 64→32 (standardized, early stopping) | Tree ensembles vs neural nets on tabular data |
| Evaluation | Accuracy + **Attack Recall** + F1, confusion matrix, feature importance | For an IDS, missed attacks (recall) matter more than accuracy |

### Results: random 80/20 split

| Model | Accuracy | Attack Recall | F1 |
|-------|----------|---------------|------|
| Random Forest | **99.90%** | ~0.999 | 0.9983 |
| Neural Network (MLP, converged at epoch 36) | 99.42% | ~0.99 | 0.9900 |

![Model comparison](figures/model_comparison.png)
![Confusion matrix](figures/confusion_matrix.png)

Random Forest beat the neural network in my results. Tree models usually do better on table style data because they ask simple yes or no questions on each column. The most important features turned out to be packet timing, flow duration, and packet size statistics. All of these describe how the traffic behaves, not what is inside the packets. That is exactly why the model fails in Part 2, where the attack is hidden inside the packet content.

![Feature importance](figures/feature_importance.png)

## 🔬 Part 2 — Cross-day generalization

No random split: train and test are entirely different capture days.

| Setup | Accuracy | Attack Recall | Precision | F1 |
|-------|----------|---------------|-----------|-----|
| Within-day baseline (random 80/20) | 99.90% | 0.999 | ~1.00 | 0.9983 |
| **Exp 1:** Train Tue+Wed → Test Friday (DDoS + PortScan) | 63.98% | **0.345** | 0.996 | 0.5127 |
| **Exp 2:** Train all other days → Test Thursday (Web Attacks) | 98.66% | **0.003** | 0.154 | 0.0061 |

![Generalization gap](figures/generalization_gap.png)

### Experiment 1: new day, one related family + one unseen family

- DDoS recall was 63% because training included DoS attacks (Hulk, GoldenEye, slowloris),
  and DDoS is the distributed version of the same flood idea, so the learned patterns
  partly carried over
- PortScan recall was only 0.16%, which surprised me since I expected scans to be caught
  easily due to their unusual traffic
- The reason scans were missed: the model only learned the exact flood shapes from training,
  while a scan is the opposite shape, just one or two tiny packets that look nearly identical
  to normal short connections
- Conclusion: an attack is detected only if it is similar to a trained attack family, not
  because it looks distinctive to a human
- Precision stayed at 0.996 while recall collapsed, meaning the model fails silently by
  labeling unseen attacks as normal instead of raising false alarms

![Per-attack recall](figures/exp1_per_attack_recall.png)

### Experiment 2: unseen web attacks and the accuracy trap

- The model caught only **2 out of 648 web attacks** (recall 0.003), yet accuracy still
  shows **98.66%**
- The reason is class imbalance: the test day is 98.7% benign, so a model that labels
  almost everything as normal still gets a high accuracy score while detecting nothing
- This result showed me in practice why accuracy is the wrong metric for intrusion
  detection on imbalanced data
- The failure is structural, not a tuning problem: XSS and SQL injection hide inside the
  HTTP payload text, and flow features never look at the payload
- No amount of extra data or a bigger model can fix this, because the features simply do
  not carry the signal
- The real fix is different features, such as payload aware models or behavioural graph
  representations, which is the current research direction in this field
  
##  Key findings

1. **A plain Random Forest can fully solve within day detection of volumetric attacks**
   (99.90% on flow features), but I learned that this number means very little for real
   deployment
2. **The generalization gap is my main result:** attack recall drops from 0.999 to 0.345
   to 0.003 as the test moves from familiar campaigns, to an unseen day, to an unseen
   attack family
3. **Accuracy is the wrong metric for intrusion detection.** In Experiment 2 the model
   scored 98.66% accuracy while catching only 0.3% of the attacks, so I report attack
   recall and F1 throughout the project
4. **What decides transfer is similarity to a trained attack family, not how unusual the
   attack looks.** The PortScan result proved my own expectation wrong, and I kept this
   negative result in the analysis instead of hiding it
5. **Flow features are blind to content based attacks by design**, since they never look
   inside the packets. This points toward payload aware and behavioural graph approaches
   rather than more data or bigger models
   
## Assumptions and limitations

- **Binary framing.** I collapsed all attack types into a single ATTACK=1 label, so the
  model only decides attack or benign. Identifying the specific attack type is future work
- **Flow level features only.** All 78 features describe traffic behaviour such as timing,
  sizes and flags. I never inspect the payload or raw packet bytes, so the model cannot
  see the content inside packets
- **Subsampling.** I randomly sampled 50k rows per CSV (with a fixed seed) to keep the
  project manageable. This under represents rare classes, for example the Thursday test
  file contains only 7 SQL Injection samples, so recall numbers on small classes are noisy
- **Single lab environment.** Even my cross day experiments stay inside CIC-IDS2017, which
  is one network, one week, and simulated users. Real deployment on a different network
  would be harder still. Testing on CSE-CIC-IDS2018 is the natural next step
- **Known dataset issues.** Published work (Engelen et al., 2021; Lanvin et al., 2023) has
  documented labelling and flow construction problems in CIC-IDS2017, so results on the
  corrected releases of the dataset can differ
- **No hyperparameter tuning.** I kept default model settings with fixed seeds on purpose,
  so that my comparisons measure the effect of the data splits and not tuning effort
  
## 📚 Dataset & citation

CIC-IDS2017 — Canadian Institute for Cybersecurity, University of New Brunswick.
Download instructions in [`data/README.md`](data/README.md). If you use the dataset, cite:

> Sharafaldin, I., Habibi Lashkari, A., & Ghorbani, A. A. (2018). *Toward Generating a New
> Intrusion Detection Dataset and Intrusion Traffic Characterization.* ICISSP 2018.

## License

MIT — see [LICENSE](LICENSE).
