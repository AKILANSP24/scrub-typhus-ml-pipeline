<div align="center">

<img src="https://upload.wikimedia.org/wikipedia/en/thumb/8/8e/Vellore_Institute_of_Technology_seal_2017.svg/200px-Vellore_Institute_of_Technology_seal_2017.svg.png" width="80" alt="VIT Logo"/>

# Host-Pathogen Detection and Severity Estimation of Scrub Typhus

### Using Genomic Data and Explainable Machine Learning

[![Python](https://img.shields.io/badge/Python-3.10-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![XGBoost](https://img.shields.io/badge/XGBoost-Latest-FF6600?style=flat-square)](https://xgboost.readthedocs.io)
[![SHAP](https://img.shields.io/badge/SHAP-Explainability-00C49F?style=flat-square)](https://shap.readthedocs.io)
[![Bowtie2](https://img.shields.io/badge/Bowtie2-2.5.5-4CAF50?style=flat-square)](http://bowtie-bio.sourceforge.net/bowtie2)
[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)
[![VIT Chennai](https://img.shields.io/badge/VIT-Chennai-8B0000?style=flat-square)](https://chennai.vit.ac.in)

**Bioinformatics · CSE4067 · J Component Project**

**School of Computer Science and Engineering, VIT Chennai**

*Winter Semester 2025–26 · April 2026*

---

[Overview](#-overview) · [Pipeline](#-pipeline-architecture) · [Results](#-results) · [Installation](#-installation--setup) · [Usage](#-usage) · [Dataset](#-dataset) · [Team](#-team)

</div>

---

## 🧬 Overview

Scrub typhus, caused by *Orientia tsutsugamushi* and transmitted by chigger mites, kills **2–30% of untreated patients** across South and Southeast Asia. It is clinically indistinguishable from dengue, malaria, leptospirosis, and typhoid at the point of hospital admission — all five diseases produce identical fever presentations. Wrong diagnosis means wrong treatment.

Even when scrub typhus is correctly identified, no published clinical tool predicts which organ — kidney, liver, or cardiovascular system — will fail first. Acute kidney injury, hepatic dysfunction, and ICU escalation each require entirely different management, yet clinicians currently have no early-warning system for these outcomes.

This project presents the **first fully reproducible, end-to-end explainable ML pipeline for scrub typhus** that addresses both challenges simultaneously:

- 🔬 **Genomic detection** using real NCBI SRA patient sequencing data
- 🏥 **5-disease clinical differential diagnosis** from routine lab values
- 🫀 **Organ-specific complication risk** — AKI, Hepatic, ICU independently modelled
- 💡 **SHAP explainability at every stage** — from raw sequencing to clinical alert

---

## 🏗 Pipeline Architecture

The system operates through four connected stages:

```
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 0 — BIOINFORMATICS PIPELINE                              │
│  NCBI SRA → FastQC → Trimmomatic → Bowtie2 (Human chr1 filter) │
│           → Bowtie2 (Orientia alignment) → SAMtools Coverage    │
│  Output: 7 genomic features per sample                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  STAGE 1A — GENOMIC DETECTION + SEVERITY SCORE                  │
│  Binary: Logistic Regression + LOO-CV → 89% accuracy            │
│  3-Class: Healthy / Convalescent / Infected                     │
│  Severity: Ridge Regression → continuous score 0.0 → 1.0        │
│  SHAP: LinearExplainer (waterfall, beeswarm, bar)               │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  STAGE 1B — CLINICAL DIFFERENTIAL DIAGNOSIS                     │
│  XGBoost + SMOTE | 2,710 patients (77.9% real data)             │
│  Classes: Scrub Typhus · Dengue · Malaria · Lepto · Typhoid     │
│  Result: 97% accuracy | F1 macro = 0.973 | AUC 0.997–1.000      │
│  SHAP: TreeExplainer (global importance, beeswarm)              │
└────────────────────────┬────────────────────────────────────────┘
                         │ (Activates when ST probability > threshold)
┌────────────────────────▼────────────────────────────────────────┐
│  STAGE 2 — ORGAN COMPLICATION RISK PREDICTION                   │
│  Three independent Random Forest models + SMOTE                 │
│  ├── AKI Risk:     AUC = 1.0000  (Top SHAP: Creatinine)         │
│  ├── Hepatic Risk: AUC = 0.9999  (Top SHAP: AST)                │
│  └── ICU Risk:     AUC = 0.9691  (Top SHAP: CRP)                │
│  SHAP: LinearExplainer per organ model                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📊 Results

### Stage 1A — Genomic Detection

| Metric | Value | Note |
|--------|-------|------|
| Overall Accuracy | **89%** | LOO Cross-Validation (n=19) |
| F1 Score (Binary) | **0.917** | Scrub Typhus vs All Others |
| Scrub Typhus Recall | **100%** | Zero infected samples missed |
| Model | Logistic Regression | Best of LR, RF, XGBoost, LightGBM |
| Validation | Leave-One-Out CV | Statistically correct for n=19 |

**Orientia Alignment Signal:**

| Label | Log Ratio Range | Genome Coverage |
|-------|-----------------|-----------------|
| Scrub Typhus (infected) | 8.93 – 10.95 | 0.176 – 0.299 |
| Convalescent | 6.97 – 9.69 | 0.102 – 0.267 |
| Dengue | 4.62 (near zero) | 0.017 |
| Malaria | 0.00 | 0.000 |
| Healthy | 0.00 | 0.000 |

### Stage 1B — Clinical Differential Diagnosis

| Disease | F1 Before | F1 After | AUC After |
|---------|-----------|----------|-----------|
| Dengue | 0.82 | **1.00** | 1.0000 |
| Leptospirosis | 0.88 | **0.97** | 0.9984 |
| Malaria | 0.72 | **0.97** | 0.9985 |
| Scrub Typhus | 0.82 | **0.97** | 0.9982 |
| Typhoid | 0.74 | **0.96** | 0.9976 |
| **Overall** | **0.795** | **0.973** | **0.997+** |

### Stage 2 — Organ Complication Risk

| Organ Model | AUC Before SMOTE | AUC After SMOTE | Top SHAP Feature |
|-------------|------------------|-----------------|------------------|
| AKI Risk | 0.9197 | **1.0000** | Creatinine (+2.60) |
| Hepatic Risk | 0.9405 | **0.9999** | AST (+2.60) |
| ICU Escalation | 0.7252 | **0.9691** | CRP (+0.97) |

---

## 🗂 Repository Structure

```
scrub-typhus-ml-pipeline/
│
├── data/
│   ├── processed/
│   │   ├── genomic_features_final.csv      # 7 genomic features, 19 samples
│   │   └── stage1b_combined.csv            # 2,710 clinical patients
│   ├── clinical_raw/
│   │   ├── Dengue Fever Hematological Dataset.csv   # Mendeley — 1,523 patients
│   │   └── Malaria Diseases dataset.csv             # Mendeley — 2,191 patients
│   └── DATA_SOURCES.md                     # All SRA accession numbers
│
├── notebooks/
│   ├── stage1_ml.ipynb                     # Stage 0, 1A, initial 1B
│   └── Adjustment_Stage.ipynb              # Real data integration, Stage 2
│
├── models/                                 # 21 trained .pkl model files
│   ├── final_binary.pkl                    # Stage 1A binary classifier
│   ├── final_multiclass.pkl                # Stage 1A 3-class classifier
│   ├── final_scaler.pkl
│   ├── severity_ridge.pkl                  # Stage 1A severity regression
│   ├── stage1b_xgb_improved.pkl            # Stage 1B XGBoost
│   ├── stage1b_scaler_improved.pkl
│   ├── stage1b_encoder_improved.pkl
│   ├── stage2_aki_improved.pkl             # Stage 2 AKI model
│   ├── stage2_aki_scaler_improved.pkl
│   ├── stage2_hepatic_improved.pkl         # Stage 2 Hepatic model
│   ├── stage2_hepatic_scaler_improved.pkl
│   ├── stage2_icu_improved.pkl             # Stage 2 ICU model
│   └── stage2_icu_scaler_improved.pkl
│
├── outputs/                                # 24 result plot PNG files
│   ├── confusion_final_binary.png
│   ├── confusion_final_multi.png
│   ├── shap_final_importance.png
│   ├── shap_final_beeswarm.png
│   ├── shap_final_waterfall.png
│   ├── severity_regression.png
│   ├── confusion_stage1b_improved.png
│   ├── roc_stage1b_improved.png
│   ├── improvement_comparison.png
│   ├── shap_stage2_aki_improved.png
│   ├── shap_stage2_hepatic_improved.png
│   └── shap_stage2_icu_improved.png
│
├── results/
│   └── alignments/                         # Bowtie2 stats per sample (.txt)
│
└── README.md
```

---

## ⚙ Installation & Setup

### Prerequisites

- Ubuntu 22.04 LTS (or WSL2 on Windows)
- Miniconda or Anaconda
- At least 8GB free disk space

### Step 1 — Create Conda Environment

```bash
conda create -n scrub_project python=3.10
conda activate scrub_project
```

### Step 2 — Install Bioinformatics Tools

```bash
conda install -c bioconda fastqc=0.12.1 -y
conda install -c bioconda trimmomatic=0.40 -y
conda install -c bioconda bowtie2=2.5.5 -y
conda install -c bioconda samtools=1.23.1 -y
conda install -c bioconda sra-tools=3.0.3 -y
conda install -c bioconda entrez-direct -y
```

### Step 3 — Install Python Libraries

```bash
pip install xgboost lightgbm scikit-learn imbalanced-learn shap \
            pandas numpy matplotlib seaborn joblib jupyter
```

### Step 4 — Download Reference Genomes

```bash
mkdir -p ~/scrub_project/genomes

# Orientia tsutsugamushi Boryong genome
datasets download genome accession GCF_000008485.1 \
    --include genome \
    --filename orientia.zip
unzip orientia.zip -d ~/scrub_project/genomes/

# Human chr1 (GRCh38)
wget -O ~/scrub_project/genomes/human_chr1.fa.gz \
    "https://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/000/001/405/GCA_000001405.15_GRCh38/GCA_000001405.15_GRCh38_assembly_structure/Primary_Assembly/assembled_chromosomes/FASTA/chr1.fna.gz"
```

### Step 5 — Build Bowtie2 Indexes

```bash
mkdir -p ~/scrub_project/indexes

bowtie2-build ~/scrub_project/genomes/orientia.fna \
    ~/scrub_project/indexes/orientia

bowtie2-build ~/scrub_project/genomes/human_chr1.fa \
    ~/scrub_project/indexes/human_chr1
```

---

## 🚀 Usage

### Run Stage 0 — Bioinformatics Pipeline

Download SRA samples (see `data/DATA_SOURCES.md` for all accession numbers):

```bash
conda activate scrub_project
cd ~/scrub_project/data/raw

# Download
for SRR in SRR6058497 SRR6058498 SRR6058499 SRR6058500 SRR6058501 \
           SRR6058502 SRR6058503 SRR6784671; do
    prefetch $SRR --output-directory ~/scrub_project/data/raw/
done

# Convert to FASTQ
for SRR in SRR6058497 SRR6058498 SRR6058499 SRR6058500 SRR6058501 \
           SRR6058502 SRR6058503 SRR6784671; do
    fasterq-dump ~/scrub_project/data/raw/$SRR \
        --outdir ~/scrub_project/data/raw/ --threads 4
done

# Trim
for SRR in SRR6058497 SRR6058498 SRR6058499 SRR6058500 SRR6058501 \
           SRR6058502 SRR6058503 SRR6784671; do
    trimmomatic SE ~/scrub_project/data/raw/${SRR}_1.fastq \
        ~/scrub_project/data/trimmed/${SRR}_trimmed.fastq \
        TRAILING:20 MINLEN:20 -threads 4
done
```

Then follow the same pattern for alignment (see Jupyter notebooks for complete code).

### Run ML Pipeline

```bash
conda activate scrub_project
cd ~/scrub_project
jupyter notebook --no-browser --port=8888
```

Open the URL shown in your browser and run:
1. `notebooks/stage1_ml.ipynb` — Stage 0 features + Stage 1A + initial Stage 1B
2. `notebooks/Adjustment_Stage.ipynb` — Real data integration + improved Stage 1B + Stage 2

### Load a Saved Model

```python
import joblib
import pandas as pd

# Load Stage 1A binary detector
scaler = joblib.load('models/final_scaler.pkl')
model  = joblib.load('models/final_binary.pkl')

# Input: 7 genomic features for a new sample
features = ['ratio', 'log_ratio', 'density', 'pathogen_reads',
            'multi_map_ratio', 'genome_coverage', 'mean_depth']
sample = pd.DataFrame([[0.047, 10.76, 47075.3, 2758, 1.0, 0.285, 0.082]],
                      columns=features)

X_scaled = scaler.transform(sample)
prediction = model.predict(X_scaled)
print("Scrub Typhus Positive" if prediction[0] == 1 else "Negative")
```

---

## 📦 Dataset

### Genomic Data (NCBI SRA)

| BioProject | Samples Used | Disease | Label |
|------------|-------------|---------|-------|
| PRJNA408161 | SRR6058497–502, SRR6058490–493, SRR6058496 | *O. tsutsugamushi* patient blood | Scrub Typhus |
| PRJNA408161 | SRR6058503, SRR6784671 | Convalescent patient blood | Convalescent |
| PRJNA799671 | SRR17752566–568 | Healthy donor blood (50k reads) | Healthy |
| PRJNA589654 | SRR10448859, SRR10448861 | P. falciparum infected blood | Malaria |
| PRJNA1071729 | SRR27828981 | Dengue patient blood (50k reads) | Dengue |

### Clinical Data (Mendeley)

| Dataset | Patients | Source | License |
|---------|----------|--------|---------|
| Dengue Fever Hematological Dataset | 1,523 | Jamalpur Hospital, Bangladesh 2024 | CC BY 4.0 |
| Malaria-Associated Clinical Data | 2,191 | Bangladesh multi-centre | CC BY 4.0 |

> **Note:** Scrub typhus, leptospirosis, and typhoid clinical records were synthetically generated from published mean±SD values across the 15 reviewed papers, as no public patient-level datasets exist for these diseases.

### Reference Genomes

| Genome | Accession | Size |
|--------|-----------|------|
| *Orientia tsutsugamushi* Boryong | GCF_000008485.1 | ~2.1 MB |
| *Homo sapiens* GRCh38 chr1 | — | ~66 MB compressed |

---

## 🔬 Research Gaps Addressed

This project directly addresses 6 gaps identified from reviewing 15 papers (2013–2025):

| Gap | How Addressed |
|-----|---------------|
| No organ-specific prediction | Stage 2: AKI, Hepatic, ICU modelled independently |
| Diagnosis and severity disconnected | Stage 1A → 1B → 2 integrated pipeline |
| No SHAP in scrub typhus ML | SHAP applied at all 3 stages |
| No publicly reproducible data | NCBI SRA + Mendeley + all 21 models saved |
| No deployed clinical tool | Complete pipeline with saved artefacts |
| Multi-class accuracy only 55–60% | 97% accuracy with real data + SMOTE |

---

## 🛠 Tools & Versions

| Category | Tool | Version |
|----------|------|---------|
| OS | Ubuntu | 22.04 LTS (WSL2) |
| Language | Python | 3.10 |
| Quality Control | FastQC | 0.12.1 |
| Trimming | Trimmomatic | 0.40 |
| Alignment | Bowtie2 | 2.5.5 |
| Coverage | SAMtools | 1.23.1 |
| SRA Download | sra-tools | 3.0.3 |
| Classification | XGBoost | Latest |
| Explainability | SHAP | Latest |
| Balancing | imbalanced-learn | Latest |
| ML Framework | scikit-learn | Latest |
| Notebooks | Jupyter | Latest |

---

## 👥 Team

<table>
<tr>
<td align="center" width="50%">
<b>Rishikesh M Ramasubramaniyan</b><br/>
Reg. No: 22MIA1163<br/>
Integrated M.Tech — Computer Science Engineering<br/>
Specialisation: Business Analytics<br/>
VIT Chennai<br/>
<a href="https://github.com/22MIA1163">GitHub</a>
</td>
<td align="center" width="50%">
<b>Akilan S P</b><br/>
Reg. No: 22MIA1191<br/>
Integrated M.Tech — Computer Science Engineering<br/>
Specialisation: Business Analytics<br/>
VIT Chennai<br/>
<a href="https://github.com/AKILANSP24">GitHub</a>
</td>
</tr>
</table>

**Faculty Supervisor**

Dr. Softya Sebastian
Assistant Professor, SCOPE
VIT Chennai — School of Computer Science and Engineering
Vandalur–Kelambakkam Road, Chennai – 600 127

---

## 📄 License

This project is released under the MIT License. See [LICENSE](LICENSE) for details.

The genomic data from NCBI SRA is publicly available under open access. The clinical datasets from Mendeley are released under CC BY 4.0.

---

## 📚 Citation

If you use this pipeline or any part of this work, please cite:

```
Rishikesh M Ramasubramaniyan and Akilan S P (2026).
Host-Pathogen Detection and Severity Estimation of Scrub Typhus
Using Genomic Data and Explainable Machine Learning.
Bioinformatics — CSE4067 J Component Project Report.
VIT Chennai, April 2026.
```

---

<div align="center">

**School of Computer Science and Engineering**

**VIT Chennai · Vandalur–Kelambakkam Road · Chennai – 600 127**

*Bioinformatics · CSE4067 · Winter Semester 2025–26*

</div>
