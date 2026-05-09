# CrispHMM
CrispHMM: a Multivariate Heterogeneous HMM to Learn Parameters of CRISPR sgRNA Predicted Efficiency

---

## Repository Contents

| File | Description |
|------|-------------|
| `binarize_chipseq.ipynb` | Data preprocessing: downloads HeLa S3 ChIP-seq BAM files from ENCODE, converts to BED, runs ChromHMM binarization at 10 kb resolution |
| `merge_features.ipynb` | Merges the ChromHMM binarized mark files (`HeLa_chrN_binary.txt`) with the sgRNA feature CSV (GC content, MFE, melting temperature, normalized efficiency) into per-chromosome merged CSVs. These are the direct inputs to the training notebook. |
| `preliminary.ipynb` | Preliminary analysis: downloads the ENCODE AWG ChromHMM segmentation for HeLa S3, intersects it with sgRNA efficiency data, runs OLS regression of chromatin state categories and proportion-active-chromatin against normalized efficiency, and computes sequence features (GC content, Hamming distances). Produces the R² = 0.006 result reported in Section 2.1 of the paper. |
| `training_model.ipynb` | Main model: defines and trains the CrispHMM HMM with mixed Bernoulli/Beta emissions and masked efficiency supervision using Pyro's Trace-ELBO |
| `viterbi_and_visualizations.ipynb` | Inference and evaluation: loads trained parameters, runs the Viterbi algorithm on held-out chromosomes, computes efficiency predictions, and generates all figures (emission/transition heatmaps, feature weight heatmap, loss curve) |
| `sgrna_features_hela.csv` | Pre-computed sgRNA features for HeLa: chromosomal coordinates, normalized efficiency, GC content, minimum free energy, and melting temperature for 8,101 guides. Direct input to `merge_features.ipynb`. |
| `sample_input/HeLa_chr21_merged.csv` | Sample input: merged ChIP-seq marks and sgRNA features for chr21 (smallest chromosome, 240,649 bins, 100 sgRNA sites) |
| `sample_output/trained_params_dl.pkl` | Sample output: trained model parameters from the 2000-step checkpoint, loadable directly into `viterbi_and_visualizations.ipynb` |

### Directory Structure

```
CrispHMM/
├── README.md
├── LICENSE
├── binarize_chipseq.ipynb
├── merge_features.ipynb
├── preliminary.ipynb
├── training_model.ipynb
├── viterbi_and_visualizations.ipynb
├── sgrna_features_hela.csv
├── sample_input/
│   └── HeLa_chr21_merged.csv
└── sample_output/
    └── trained_params_dl.pkl
```

---

## Large Data Files

These files are too large to include in the repository. Download them before running the preprocessing notebook.

**ENCODE AWG ChromHMM segmentation (HeLa S3, hg19)** — used in `preliminary.ipynb`:
- `wgEncodeAwgSegmentationChromhmmHelas3.bed.gz` — https://hgdownload.cse.ucsc.edu/goldenpath/hg19/encodeDCC/wgEncodeAwgSegmentation/

**ChIP-seq data (ENCODE, HeLa S3, hg19)** — download from UCSC ENCODE DCC:
- `wgEncodeBroadHistoneHelas3CtcfStdAlnRep1.bam` — https://hgdownload.cse.ucsc.edu/goldenPath/hg19/encodeDCC/wgEncodeBroadHistone/
- `wgEncodeBroadHistoneHelas3H3k27acStdAlnRep1.bam`
- `wgEncodeBroadHistoneHelas3H3k27me3StdAlnRep1.bam`
- `wgEncodeBroadHistoneHelas3H3k36me3StdAlnRep1.bam`
- `wgEncodeBroadHistoneHelas3H3k4me1StdAlnRep1.bam`
- `wgEncodeBroadHistoneHelas3H3k4me2StdAlnRep1.bam`
- `wgEncodeBroadHistoneHelas3H3k4me3StdAlnRep1.bam`
- `wgEncodeBroadHistoneHelas3H3k9acStdAlnRep1.bam`
- `wgEncodeBroadHistoneHelas3H4k20me1StdAlnRep1.bam`

**sgRNA efficiency data (DeepCRISPR, HeLa)**:
- `13059_2018_1459_MOESM5_ESM.xlsx` — Supplementary Table 5 from Chuai et al. 2018, *Genome Biology*. Available at: https://doi.org/10.1186/s13059-018-1459-4

**Genome chromosome sizes**:
- `hg19.chrom.sizes` — https://hgdownload.cse.ucsc.edu/goldenPath/hg19/bigZips/hg19.chrom.sizes

---

## Setup and Requirements

**Python**: 3.9 or later

**Dependencies**:
```
torch>=2.0
pyro-ppl>=1.8
pandas
numpy
scipy
matplotlib
seaborn
openpyxl
bedtools (system, via apt or conda)
samtools (system, via apt or conda)
java (for ChromHMM binarization)
```

Install Python dependencies:
```bash
pip install torch pyro-ppl pandas numpy scipy matplotlib seaborn openpyxl
```

**Hardware**: A GPU is strongly recommended for training. The model was developed and trained on Google Colab with an A100 GPU. Training on CPU is possible but slow. The Viterbi inference notebook will auto-detect CUDA, Apple MPS, or CPU.

**Memory**: Full training on all HeLa chromosomes requires ~16 GB RAM.

---

## How to Run

### Step 1 — Preprocess data
Run `binarize_chipseq.ipynb` end-to-end. This will download and binarize the ChIP-seq BAM files at 10 kb resolution using ChromHMM.

Output: a directory of binarized mark files, one per chromosome (e.g. `HeLa_chr1_binary.txt`).

### Step 1b — Merge marks with sgRNA features
Run `merge_features.ipynb`. This reads the `HeLa_chrN_binary.txt` files from Step 1 and merges them with `sgrna_features_hela.csv` (sgRNA coordinates, GC content, MFE, melting temperature, normalized efficiency) at 200 bp bin resolution, producing one `HeLa_chrN_merged.csv` per chromosome.

### Step 2 — Preliminary analysis (optional)
Run `preliminary.ipynb` to reproduce the OLS regression of chromatin state against efficiency reported in Section 2.1 of the paper.

### Step 3 — Train CrispHMM
Run `training_model.ipynb`. Update `DRIVE_PATH` and `MERGED_DIR` at the top to point to your merged CSV directory and desired output location.

Key parameters (set in the `args` namespace at the top of the notebook):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `hidden_dim` | 11 | Number of latent chromatin states |
| `chunk_size` | 1612 | Bins per forward pass through HMM |
| `num_steps` | 2000 | Training iterations |
| `lr` | 0.01 | Learning rate |

Output: `trained_params_dl.pkl` — a pickle file containing all learned parameters and training loss history.

### Step 4 — Inference and evaluation
Run `viterbi_and_visualizations.ipynb`. Set `PARAMS_PATH` to point to your saved `trained_params_dl.pkl`. This notebook:
- Runs Viterbi decoding on held-out chromosomes (chr13, chr21, chr22)
- Computes Pearson/Spearman correlation and AUC-ROC for efficiency predictions
- Generates emission matrix, transition matrix, and feature weight heatmaps

---

## Test Run on Sample Data

To verify your environment is set up correctly without running the full pipeline:

```bash
jupyter notebook viterbi_and_visualizations.ipynb
```

Point `MERGED_DIR` to `sample_input/` and `PARAMS_PATH` to `sample_output/trained_params_dl.pkl`. The notebook will run Viterbi on chr21 and produce efficiency predictions for the 100 sgRNA sites in that chromosome.

---

## Citation

If you use CrispHMM, please cite:

> Lacher, S., Tan, T., and Zeng, D. CrispHMM: a Multivariate Heterogeneous HMM to Learn Parameters of CRISPR sgRNA Predicted Efficiency. 2026.

---

## License

GPLv3. See `LICENSE` for details.
