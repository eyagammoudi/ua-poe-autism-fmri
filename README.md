# UA-PoE

**Precision-weighted Product-of-Experts fusion for cross-site autism diagnosis from rs-fMRI (ABIDE-I, strict LOSO).**

This repository contains the code to reproduce all results in:

> E. Gammoudi, V. P. Panagiotou, C. Makris.
> *Precision-weighted Product-of-Experts fusion for robust cross-site autism
> diagnosis from resting-state fMRI: a controlled comparison of multi-view
> fusion rules.* 


---

## What this is

UA-PoE represents each view of the brain as a calibrated diagonal-Gaussian
*expert* and fuses them in closed form with a precision-weighted **Product of
Experts (PoE)**. One site-robust static functional-connectivity expert
(tangent-space embedding + ComBat) is combined with three atlas-specific dynamic
FC experts encoded by Mamba state-space networks. The study isolates the **fusion
rule** as the experimental variable and evaluates it under strict
leave-one-site-out (LOSO) on ABIDE-I.

The contribution is methodological and evaluative, not a new algorithm or a
state-of-the-art accuracy. Read the results below with that framing in mind.

## Key results (LOSO, ABIDE-I, 871 subjects, 20 sites)

| Model | LOSO AUC |
|---|---|
| Logistic regression (static FC, baseline) | 0.721 |
| MLP (static FC) | 0.704 |
| Mamba–dFC (single view) | 0.521 |
| Transformer–dFC (single view) | 0.672 |
| Mixture of Experts (same experts) | 0.645 |
| UA-PoE (correlation static) | 0.710 |
| **UA-PoE (tangent + ComBat, ours)** | **0.736 ± 0.011** |

- Pooled 95% CI: **[0.710, 0.773]** — contains the linear baseline, so UA-PoE is
  **statistically on par** with logistic regression on accuracy.
- Controlled comparison: **PoE 0.710 vs. MoE 0.645** (+0.065 AUC; per-site
  0.631 vs. 0.721, paired Wilcoxon *p* = 0.049).
- Predictive uncertainty is higher on errors (uncertainty→error AUC 0.623);
  inference costs 1.28 ms/subject.

## Data

This repository does **not** redistribute any imaging data. ABIDE-I is obtained
through the Preprocessed Connectomes Project (PCP):

- Portal: http://preprocessed-connectomes-project.org/abide/
- We use the **ROI time-series derivatives** (one mean BOLD series per region),
  downloaded via `nilearn.datasets.fetch_abide_pcp`.
- Atlases: Craddock-200 (CC200, primary), AAL, Harvard–Oxford (HO), giving
  19,900 / 6,670 / 6,105 connectivity features respectively.

Working cohort after QC: 871 subjects (403 ASD, 468 controls) from 20 sites. Use
of ABIDE is subject to its own data-use terms; please cite Di Martino et al.
(2014).

## Installation

```bash
git clone https://github.com/<your-username>/ua-poe.git
cd ua-poe
python -m venv .venv && source .venv/bin/activate    # Python >= 3.10
pip install -r requirements.txt
```

Core dependencies: `numpy`, `scipy`, `scikit-learn`, `nilearn`, `torch`,
`mamba-ssm`, a ComBat implementation (`neuroHarmonize` or `neurocombat`), and
`captum` + `shap` for attribution.

> **Note on `mamba-ssm`:** the selective-scan kernel expects a CUDA GPU. On
> CPU-only machines, install the reference/fallback path (see
> `requirements-cpu.txt`) — results are identical but slower.

Pin your exact versions in `requirements.txt` before releasing, and record the
CUDA/PyTorch build used for the reported numbers so a reviewer can reproduce the
environment exactly.

## Reproducing the results

```bash
# 1. Fetch and cache ABIDE-I ROI time series (CC200, AAL, HO)
python scripts/prepare_data.py --atlases cc200 aal ho

# 2. Full leave-one-site-out sweep: 20 sites x 5 seeds
python scripts/train_loso.py --model ua_poe --seeds 0 1 2 3 4

# 3. Baselines and the fusion-rule comparison (PoE vs MoE, ablations)
python scripts/train_loso.py --model moe   --seeds 0 1 2 3 4
python scripts/baselines.py   --models lr mlp
python scripts/ablation.py    --seed 0

# 4. Interpretability: per-expert importance, attribution consensus
python scripts/explain.py --site NYU

# 5. Aggregate metrics, CIs, significance tests, and regenerate all figures/tables
python scripts/report.py
```

<!-- ACTION REQUIRED: rename the commands above to match your actual scripts. -->

Expected wall-clock: ~147 s per LOSO fold (one seed); ~246 min for the full
five-seed, twenty-fold sweep on a single GPU.

## Key hyperparameters

| Setting | Value |
|---|---|
| Latent dim per expert | 48 (diagonal Gaussian) |
| Dynamic encoder | 2 residual Mamba blocks, d_model = 32 |
| Window / stride / count | 30 / 15 / 12 |
| Monte-Carlo draws (L) | 10 |
| KL weight (β) | 1e-3 |
| Adversary cap (λ_max) | 0.05 (gradient-reversal, slow ramp) |
| Optimiser | Adam, lr 1e-3, weight decay 1e-4 |
| Batch size / epochs | 32 / 30 |
| Seeds | {0, 1, 2, 3, 4} |

All fold-specific transforms (standardisation, tangent reference, ComBat) are fit
on the training split of each LOSO fold only — no test-site labels are ever used.

## Repository structure

```
ua-poe/
├── README.md
├── requirements.txt
├── scripts/
│   ├── prepare_data.py      # fetch ABIDE-I, build static + dynamic FC
│   ├── train_loso.py        # LOSO training for ua_poe / moe
│   ├── baselines.py         # logistic regression, MLP
│   ├── ablation.py          # component ablations
│   ├── explain.py           # per-expert + saliency/IG/DeepSHAP attribution
│   └── report.py            # metrics, CIs, tests, figures, tables
├── src/
│   ├── experts.py           # Gaussian experts, Mamba encoders
│   ├── poe.py               # closed-form precision-weighted fusion
│   ├── harmonize.py         # tangent-space embedding + ComBat
│   └── data.py              # windowing, atlases, LOSO splits
└── configs/
    └── ua_poe.yaml
```

<!-- ACTION REQUIRED: make this tree match the repo you actually publish. -->

## Citation

```bibtex
@article{gammoudi2025uapoe,
  title   = {Precision-weighted Product-of-Experts fusion for robust cross-site
             autism diagnosis from resting-state fMRI: a controlled comparison
             of multi-view fusion rules},
  author  = {Gammoudi, Eya and Panagiotou, Vlasios Panagiotis and Makris, Christos},
  journal = {Computer Methods and Programs in Biomedicine},
  year    = {2025},
  note    = {Under review}
}
```

## License

<!-- ACTION REQUIRED: choose a license (e.g. MIT) and add a LICENSE file. -->
Code released under the MIT License (see `LICENSE`). ABIDE-I data are governed by
their own terms of use and are not included in this repository.

## Contact

Eya Gammoudi — eya.gammoudi@upatras.gr
