# Statistical analysis

This document summarises the statistical analysis of the experiment comparing the transpilation of Grover circuits as a whole (**Whole**) against the reuse of stored fragments (**Fragments**, and **Frag. - 1 Oracle**). It explains what was computed, how to read each output file, what the results say, and what the limitations are.

## 1. What was computed and why

All the files generated are in the `statistical_results` folder. 

| Output file | Analysis | Question it answers |
|---|---|---|
| `1_descriptivos.csv` | Mean, std, median, quartiles, 95% bootstrap CI per configuration | How much do the 10 measurements vary? Source for the paper table. |
| `2_A_whole_vs_fragments_wilcoxon.txt` | Paired Wilcoxon on configuration means: global and per number of qubits (Holm) | Is one approach faster at each circuit size? |
| `2_A_..._mannwhitney_por_configuracion.csv` | Mann-Whitney U per configuration with the A12 effect size (Holm) | Is the difference significant in each single configuration? |
| `2_B_fragments_vs_1oracle_wilcoxon.txt` | Same as A for Fragments vs Frag. - 1 Oracle, plus a one-sided test and overhead statistics | Does handling a new oracle add a cost? |
| `2_B_..._mannwhitney_por_configuracion.csv` | Same as A, per configuration | Idem, per configuration |
| `4_regressions.txt` | Regression of log2(time) on qubits (Whole) and of time on (M+2) and qubits (Fragments) | Does Whole double per qubit? Does Fragments cost scale with the number of fragments? |
| `5_ratios_ic.csv` | Ratios Whole/Fragments and Frag1Oracle/Fragments with 95% bootstrap CI | How large is the speedup or overhead, and is it distinguishable from 1? |
| `6_aer_equivalence.txt` | Aer: paired mean-error difference with a 90% CI and TOST equivalence (margin +-0.01); Wilcoxon on execution time | Do both approaches give equivalent results? |
| `6b_aer_execution time.csv` | Execution-time ratio Whole/Fragments per configuration | Where do the execution times differ? |
| `7_times_*.png` | Time vs qubits with error bars | Visual summary |

## 2. Results

### 2.1 Whole vs Fragments (transpilation time)

Paired Wilcoxon on configuration means (Holm-adjusted per size):

| Qubits | Adjusted p | Reading |
|---|---|---|
| 4 | 0.0137 | _Whole_ faster (10 of 10 pairs) |
| 5 | 0.0273 | _Whole_ faster |
| 6 | 0.0645 | No significant difference |
| 7, 8, 9, 10 | 0.0137 | _Fragments_ faster (10 of 10 pairs) |

Global test: W = 547, p = 4.7e-05; _Whole_ is faster in 27 of the 70 pairs.

Per-configuration Mann-Whitney (Holm over 70 tests), 10 configurations per size:

| Qubits | Result |
|---|---|
| 4 | _Whole_ significantly faster in 10 of 10 |
| 5 | _Whole_ faster with 8 and 10 elements (4 of 10); no significant difference otherwise |
| 6 | _Whole_ faster with 8 and 10 elements (4 of 10); no significant difference with 1, 2 and 4 |
| 7 | _Fragments_ faster with 1 and 2 elements (4 of 10); 4 elements borderline (adjusted p = 0.0509, A12 = 0.90 in both backends); not significant with 8 and 10 |
| 8, 9, 10 | _Fragments_ faster in all 30 configurations (A12 between 0.94 and 1.0) |

**Crossover**: 7 qubits. 6 qubits is a statistical tie. With 7 or more qubits, the CI of the ratio excludes 1 in favour of _Fragments_ in 38 of the 40 configurations (the exceptions are `fake_fez`, 7 qubits, 8 and 10 elements, with ratios 1.17 and 1.13).

Speedups with 10 qubits (ratio _Whole/Fragments_, 95% CI):

| Marked elements | `fake_fez` | `fake_marrakesh` |
|---|---|---|
| 1 | 11.13 [8.93, 14.50] | 11.38 [9.26, 14.68] |
| 10 | 5.41 [4.60, 6.43] | 5.43 [4.18, 7.17] |

The advantage decreases as more elements are searched but never disappears for 8 or more qubits.

### 2.2 Fragments vs Frag. - 1 Oracle (transpilation time)

- _Frag. - 1 Oracle_ is slower than _Fragments_ in **56 of 56** configurations (global Wilcoxon   W = 0, p = 7.55e-11; one-sided p = 3.77e-11).
- Overhead: median **7.4%** (Q1 3.3%, Q3 16.9%), maximum +137.7% (`fake_marrakesh`, 5 qubits, 4 elements); **13** configurations exceed 20%.
- In **38 of 56** configurations the 95% CI of the ratio includes 1, so the overhead cannot be distinguished from zero. In **18 of 56** it is above 1 (real overhead), concentrated at 4, 5
  and 8 qubits; none appears at 6 or 7 qubits. All 13 cases above 20% are among these 18.
- The per-size tests give an adjusted p of 0.0547 for every size. This is not evidence of no effect: with only 8 pairs per size and 7 sizes, 0.0547 is the best adjusted p-value possible.

### 2.3 Scaling (regressions)

- **Whole**, log2(time) vs qubits, fitted on 7 to 10 qubits (only 4 points per fit): slope 0.94 to 1.05 in `fake_fez` (x1.92 to x2.07 per qubit, R2 >= 0.996) and 0.84 to 0.95 in `fake_marrakesh` (x1.79 to x1.93, R2 >= 0.986). Time roughly doubles per qubit, slightly less in `fake_marrakesh`.
- **Fragments**, linear model: `fake_fez`: t = -0.454 + 0.075*(M+2) + 0.070*qubits (R2 = 0.892); `fake_marrakesh`: t = -0.475 + 0.084*(M+2) + 0.069*qubits (R2 = 0.889). Roughly 0.08 s per extra fragment and 0.07 s per extra qubit. The negative intercept is an artefact of the linear fit and has no physical meaning. R2 near 0.89 means the model captures most but not all of the variance; for example, _Fragments_ with 10 elements is sometimes faster than with 8, which a model based only on the number of oracles cannot reproduce. A possible (untested) reason is that the cost of an oracle depends on which states are marked and not only on how many.

### 2.4 Aer validation (35 cases)

- **Mean error**: mean difference (_Whole_ - _Fragments_) = -0.00025, 90% CI [-0.00079, 0.00029], mean |difference| = 0.00127, maximum |difference| = 0.00547; Wilcoxon p = 0.798. The CI lies inside the +-0.01 margin, so the errors are equivalent within it. The TOST p-value (1.5e-26) is
  tiny only because the margin is much wider than the CI; cite the CI.
- **Execution time on Aer**: Wilcoxon p = 0.0241; median ratio _Whole_/_Fragments_ = 1.071, range 0.689 to 2.077. The composed circuits give the same error but are not always equivalent in simulation cost. 
