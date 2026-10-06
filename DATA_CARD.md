# Repaired synthetic journal-entry benchmark (version 1.0.0)

## What this is
Counterfactual twins of the synthetic general ledger of Gronewald et al., regenerated with the authors' own generator
notebook for 20 generator seeds. Two paired versions exist for every seed; both are released for the dev seeds,
only `repaired` for the test seeds (see "Evaluation rules"):
- `repaired` (twin `Iv_Cp_Ae`): the identity and chain-inheritance side effects of the anomaly injection are removed;
  every injected mechanism is unchanged.
- `published_generator` (twin `Ic_Cb_Ae`): the authors' injection behaviour (for seed 42 it reproduces the published file
  except 214 `top_n` cells that are ties at the per-year rank-1000 boundary).
The two versions of a seed share posting IDs, row order, labels, markings and chains and differ only in `user`, so the
difference in a detector's score between them measures the side effects.

## Evaluation rules (required for valid results)
- Score each version on its own. A method must not use the other version of the same seed in any way (joining, diffing,
  training on one and scoring the other, domain adaptation): the two versions differ only on anomalous chains, so a diff
  of them flags 85% of the anomalous chains of the test seeds with 0 false
  positives (`version_diff_check.csv`).
- Do not regenerate any seed with the authors' public generator: the seeds are disclosed, and the published-generator
  version of a seed paired with its repaired version reveals the anomalies in the same way.
- Design and tune on the dev seeds; report on the test seeds. For this reason the `published_generator` version of the
  test seeds is withheld; it is released for the dev seeds, where paired side-effect analyses are intended.
- Labels (`entries`, `chains`, `injections`) are for evaluation only and must never be a method input. `injections`
  together with `entries` also allows the withheld test-seed `published_generator` version to be reconstructed.
- `baselines/` holds a label-free reference score built from facts in the authors' generator notebook. Report it as a
  reference row; a method that uses it as an input must say so, because it then uses knowledge of the generator.

## Source and licence
- Source article: J. Gronewald, S. Stephan, A. Rombach, P. Fettke, "Generating synthetic journal entries for audit
  analytics: Method and dataset", Data in Brief 69 (2026) 113257, https://doi.org/10.1016/j.dib.2026.113257.
- Source data and generator notebook: J. Gronewald, S. Stephan, A. Rombach, P. Fettke, "DIB General Ledger Data",
  Zenodo (2026), https://doi.org/10.5281/zenodo.22014653, licensed CC BY 4.0
  (https://creativecommons.org/licenses/by/4.0/).
- This derivative is released under CC BY 4.0 with no additional restrictions. The material was modified; the changes
  are listed below. The original authors do not endorse this derivative.
- Provided as is, without warranties of any kind (see CC BY 4.0, Section 5).

## Statement of changes (relative to the authors' generator)
1. Identity. The six hard-coded injected user names (markings 1-6: Max, Fred, Fred, David, Max, Fred) are replaced in
   the `repaired` version by a user drawn per (chain, published name) from the users with the SAME pool status (inside /
   outside the posting's scenario pool) as the published name, never the posting's original preparer. Where the
   published code left the preparer unchanged (published name = original preparer), nothing is changed. Separate RNG
   stream: random.Random(seed + 1000). The published name has the same pool status as itself, so it remains one of the
   candidates: it is drawn again for 39% of the injected postings whose preparer the published code changed.
   What the repair removes is the constant name per marking, not every occurrence of that name.
2. Chain inheritance. The authors change the preparer of the injected posting only, which breaks the generator's rule
   that clearings inherit the invoice's preparer. In the `repaired` version the new preparer is copied to the normal
   (label 0) postings of the same invoice/clearing chain, using the final links after the cross-linked clearing
   injection (marking 8). These partner postings keep label 0 and are flagged `role = partner`.
3. Account 1200 is NOT changed in the released versions (it remains exclusive to marking 7, as in the authors'
   generator; the study treats a reroute of normal postings to 1200 as a mechanism ablation, not a side effect).
4. Order of operations: JET injections (markings 1-7) -> cross-linked clearing (marking 8) -> chain propagation.
5. Release processing: posting IDs replaced by random opaque IDs (consistently in `ref_id` and across the two versions of
   a seed); rows shuffled across postings (lines of a posting stay together, in their original order); labels moved to
   separate files; `top_n_tie_inclusive` added. Generator RNG for these steps: numpy default_rng(seed + 9000).

## Files
- All tables are Apache Parquet (ledgers zstd-compressed; readable with pyarrow, polars or the R arrow package) except
  the CSV files named below.
- `ledger/seed{S}_{version}.parquet` (`published_generator` for dev seeds only): line level; columns posting_id, document_date, posting_date, entered_date, gl_account, gl_account_name, amount, cd_flag, user, tax_rate, ref_id, text, promptly, weekend, nwh, top_n, high_cash, top_n_tie_inclusive. No labels.
- `entries/seed{S}_{version}.parquet`: one row per posting: posting_id, chain_id, label, marking, role
  (injected / partner / normal), is_partner_of_anomaly.
- `chains/seed{S}_{version}.parquet`: chain_id, n_entries, chain_label, chain_markings (every marking in the chain,
  ';'-separated), n_injected_entries, n_partner_entries.
- `injections/seed{S}_{version}.csv`: every injected preparer assignment; columns posting_id, published_user,
  original_user, new_user.
- `baselines/seed{S}_{version}_documented_facts_rule.parquet`: a label-free reference rule built only from facts written
  in the authors' generator notebook (documented user pools per account pattern, account 1200, category-mismatched
  links, cash on invoices, JET flags, multi-preparer chains). It is a reference point, not a detector.
- `splits.json` (dev seeds 42-51, test seeds 52-61), `release_checks.csv` (integrity and
  leak checks), `release_shortcut_screen.csv` (single-feature screen), `version_diff_check.csv` (what a version diff
  reveals; the reason the test-seed `published_generator` files are withheld), `SHA256SUMS`.

## Unit of evaluation
Evaluate at the chain level (invoice plus its clearings, `chain_id`). Recommended primary metric: footprint-fixed chain
AUROC / AUPRC, i.e. an anomalous chain is scored by its injected postings only and a normal chain by all its postings.
Reason: `role = partner` marks the normal-labelled members of anomalous chains (on average
831 entries per seed). In the `repaired` version these partners carry the
injected user wherever the injection changed the preparer, and some of them fall inside the JET population of their
chain's marking; an entry-level evaluation would count them as false positives by construction, and a chain-max
evaluation would give anomalous chains a second high-scoring entry. Report per marking.

## Per-marking ceilings and known non-identifiability
Oracle ceilings (footprint-fixed chain AUROC / AUPRC, mean over seeds; scikit-learn definitions: tied scores count one
half in AUROC, and AUPRC is the tie-grouped average precision, as for every scorer in the study). The oracle knows each
marking's JET population and the documented mechanism facts (scenario user pools, cash on invoices, account 1200,
category-mismatched links) and uses the generator's scenario of each posting, which is NOT in the release; it uses no
side effect, so it is the same for both versions and is an upper bound on what the released fields allow:

| marking | published_generator | repaired |
|---|---|---|
| m1 | 0.989 / 0.811 | 0.989 / 0.811 |
| m2 | 0.998 / 0.227 | 0.998 / 0.227 |
| m3 | 0.988 / 0.761 | 0.988 / 0.761 |
| m4 | 1.000 / 0.938 | 1.000 / 0.938 |
| m5 | 1.000 / 1.000 | 1.000 / 1.000 |
| m6 | 0.963 / 0.753 | 0.963 / 0.753 |
| m7 | 1.000 / 1.000 | 1.000 / 1.000 |
| m8 | 1.000 / 0.952 | 1.000 / 0.952 |

- Marking 2 has no "unusual user" mechanism: its published injected user (Fred) is a regular user of the scenario in
  which every marking-2 posting lies. In the published version it is detectable mainly through the side effects; in the
  repaired version it is close to label noise inside its JET population. Treat it as non-identifiable.
- Markings 1, 3, 4 and 6: part of the injected users are regular users of their scenario (the authors' design); the
  ceilings above reflect this.

## Shortcut screen of the released files
Single-feature AUROC per marking (entry level, partners excluded, categoricals out-of-fold target-encoded by chain
groups) on test seeds 52-61: no feature is materially more separating in `repaired` than in
`published_generator` (largest increase over all features and markings +0.0025, for
ref_user_mismatch / marking 8). Features above 0.9 on `repaired` for some marking
(acct_sig, chain_size, delay, high_cash, log_amount, nwh, promptly, text, top_n, top_n_ti, user_x_sig, user_x_text, weekday, weekend) are mechanism, JET-population or documented
user-pool features; their largest increase over `published_generator` is +0.0000. Full table in `release_shortcut_screen.csv`.

Posting-ID number and row position carry no label information: both come from random permutations drawn independently
of the data (statement of changes, item 5). Over all postings of each `repaired` ledger, the AUROC
of the label against the posting-ID number is 0.487-0.519 and against the row position
0.474-0.515 (20 seeds; `release_checks.csv`). The screen above folds every AUROC to max(a, 1 - a),
which lies above 0.5 even without signal; there the two features reach 0.506-0.540 per marking, against
0.508-0.552 expected with no signal for the number of anomalies of each marking (normal approximation). The 18 differences from that expectation lie between -2.3 and +2.4 standard errors of a 10-seed mean (7 above, 11 below), as expected from noise.

## Known residual artefacts (present in both versions, part of the authors' mechanisms or design)
- Marking 8 (cross-linked clearing): the re-pointed invoice is paid twice, the original invoice is left unpaid, and the
  clearing's preparer differs from its new invoice's preparer. Chain size and number of clearings separate marking 8
  almost perfectly.
- Some anomalous chains still have more than one preparer in the repaired version (chains with several anomalies of
  different markings, marking-8 rewiring): see `chains` and the structure table below.
- `top_n` is arbitrary for postings tied at the rank-1000 boundary (the authors' non-stable sort);
  `top_n_tie_inclusive` is deterministic. Differences per seed: 37-224 postings.
- JET scope flags are the populations the anomalies are sampled from; a count of flags alone is a strong baseline.
- Generator facts a detector can exploit: documented user pools per scenario, account 1200 for marking 7. The scenario
  itself (formerly encoded in the posting-ID prefix, number and row order) is not released.

## Structure (per seed; mean, min, max over seeds)
```
version                              published_generator  repaired
n_entries                      mean             102413.0  102413.0
                               min              102384.0  102384.0
                               max              102452.0  102452.0
n_chains                       mean              51579.0   51579.0
                               min               51579.0   51579.0
                               max               51579.0   51579.0
n_anom_chains                  mean                832.6     832.6
                               min                 823.0     823.0
                               max                 841.0     841.0
n_partner_entries              mean                831.2     831.2
                               min                 811.0     811.0
                               max                 850.0     850.0
partner_entries_in_marking_pop mean                216.2     216.2
                               min                 197.0     197.0
                               max                 233.0     233.0
multi_marking_chains           mean                  7.4       7.4
                               min                   2.0       2.0
                               max                  13.0      13.0
multi_user_anom_chains         mean                710.8      21.4
                               min                 694.0      16.0
                               max                 724.0      25.0
multi_user_normal_chains       mean                  3.8       3.8
                               min                   1.0       1.0
                               max                   7.0       7.0
normal_entries_on_1200         mean                  0.0       0.0
                               min                   0.0       0.0
                               max                   0.0       0.0
```

## Reproducibility
Built by `09_release` from artefacts of `01_decomposition` in the Docker image recorded in `docker/image_id.txt`
(sha256:2c2170ce9c0a88b3bf7d4b9730fba5b2657d7a9e5ee85da6caecaf2249288731 2026-10-05T16:08:29.856081592Z arm64; linux/arm64). The released files, not a rebuild, are canonical: the platform-dependent `top_n` tie order is one
known source of difference on other platforms. The generator patch is described exactly in the statement of changes;
the study's code is not part of this release. `release_checks.csv`, `release_shortcut_screen.csv` and
`version_diff_check.csv` were computed on both versions of every seed before the test-seed `published_generator` files
were withheld; from the release itself, the screen's `published_generator` column and the version-diff check can be
reproduced only in their dev-seed form.

## Intended use and limitations
Benchmarking ledger anomaly detectors on synthetic data with and without the injection side effects. Synthetic only:
no claim is made about real ledgers, real fraud prevalence (here about 0.8% of entries and 1.6% of chains), or real
preparer behaviour.

## How to cite
Cite this dataset, version 1.0.0, as https://doi.org/10.5281/zenodo.23188844 (the DOI becomes active when the dataset is
published together with the article that introduces it), and cite that article and the source data above. The article
reference will be added here on publication.
