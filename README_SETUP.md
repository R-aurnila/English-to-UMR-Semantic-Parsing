# Replicating and Extending SETUP: English Text → UMR Parsing

Replication of **SETUP: Sentence-level English-To-Uniform Meaning Representation Parser** (Markle, Gutierrez Bach & Wein, LREC 2026), plus two extensions.

**Task:** English sentence in → UMR meaning graph out.

1. **Job 1:** AMRBART parser rebuilt from scratch; scores compared with the paper. No AMRBART code in the authors' repo (only BiBL and T5), so written myself.
2. **Job 2:** Modern open LLM (Qwen3-8B) as a parser.
3. **Job 3:** Attempted fix for the paper's main weakness: normal English sentences (non minecraft).

---

## Data

- **UMR v2.0, English only**, as in the paper (§3.3)
- ~84% **Minecraft game chat** (e.g. `[Builder puts down a blue block at X:1 Y:2 Z:3]`), the rest normal English (news, stories)
- All results split into **Minecraft vs. non-Minecraft**, same rule as the paper (§4.1)
- **66 overlap graphs not removed:** paper excludes 66 graphs that overlap with AMR training sentences (§3.4); kept here for simplicity (66 of ~30k sentences)

### Two splits

- **Split A (paper-style):** authors' dev set (from their repo) --> random test set of the paper's size (2,941). Paper's own test set not public. This is the closest I could get to the paper.
- **Split B (clean):** split by document, exact duplicates removed from dev/test. Reason: **~55% of split-A test sentences also in train**, word for word match (mostly repeated Minecraft lines).

---

## Job 1: AMRBART replication

### Main choices

- **Start point:** public AMRBART parser (trained on AMR 3.0), fine-tuned on UMR (§3.2)
- **Training:** 10 epochs, lr 4e-5 (paper); batch 16, warmup, label smoothing, beam 5 (AMRBART defaults, not in paper)
- **Best epoch:** SMATCH on 1,000 dev sentences
- **Metrics:** AnCast, SMATCH, SMATCH++ (§3.5)

### Issues found

1. **AMRBART's decoder drops UMR info** (`:aspect state`, `:refer-number plural`). Fixed decoder written. Test: gold graphs encoded and decoded back, no model involved (= best score any trained model could reach). AMRBART's decoder: **98.7 AnCast**; fixed decoder: **99.6**.
2. **AnCast pitfalls:** graphs given as one text string → only the first sentence scored; So they are sent as file paths. "AMR mode" → UMR-only parts (`:aspect`, `:refer-number`) ignored, so missing them still scores 100. All scores here: graphs given as files, UMR mode.
3. **Own decoder bug:** numbers glued to roles (`:quant33` instead of `:quant 33`). Fixed; effect ≤0.3 points.

### Results (test)

| Model | AnCast | SMATCH | SMATCH++ |
|---|---|---|---|
| Paper, AMRBART | 81.7 | 86.2 | 88.7 |
| Paper, BiBL (their best) | 84.4 | 88.8 | 91.0 |
| **Mine, AMRBART, split A** | **86.9** | **91.1** | **91.2** |
| Mine, AMRBART, split B (clean) | 79.4 | 86.2 | 86.2 |

**Above the paper.** Likely reasons: fixed decoder keeps UMR info; different test sample with many train duplicates.

### Minecraft vs. normal English (new: paper shows this only for its baseline)

| AnCast | Minecraft | Non-Minecraft |
|---|---|---|
| Split A | 92.6 | **66.8** |
| Split B (clean) | 84.2 | **68.4** |

- **~26 points worse on normal English**
- Clean split: Minecraft −8, non-Minecraft unchanged → **duplicates inflate the paper-style score**; real weakness = normal English

---

## Job 2: Modern LLM as parser

**Qwen3-8B-Base + LoRA**, same data, same scoring.

### Main choices

- **Graph format:** one line, variables renamed `z0, z1, …` (annotator names like `s34a` = noise)
- **Training:** LoRA rank 16 on all layers, lr 2e-4, 3 epochs (common LoRA defaults; no LLM parser in paper)
- **Decoding:** greedy, vs. AMRBART's beam 5 (slight advantage to AMRBART)

### Results (AnCast)

| | All | Minecraft | Non-Minecraft |
|---|---|---|---|
| AMRBART, split A | 86.9 | 92.6 | 66.8 |
| Qwen3-8B, split A | 84.0 | 91.9 | **57.0** |
| AMRBART, split B | 79.4 | 84.2 | 68.4 |
| Qwen3-8B, split B | 75.5 | 82.5 | **59.7** |

- Game chat: near tie
- Normal English: **9–10 points worse**, both splits
- Likely reason: AMRBART's prior AMR training helps most on normal English (closest to AMR data); Qwen3 never saw meaning graphs. Supports the paper's build-on-AMR approach (§3.2).

---

## Job 3: Fixing the normal-English weakness

**Paper's explanation (§4.2):** too few normal-English sentences in training. Tested directly with AMRBART.

### Methods

- **R1 Upsample:** each normal-English sentence repeated 4× (3,415 → 13,660 examples; share of training 16% → 43%)
- **R2 Two-stage:** Job 1 model + extra training on normal English only, small lr
- **R3 Combined:** R1, then R2

### Evaluation

- **Best epoch:** dev score weighting Minecraft and normal English equally
- **Winner:** picked on dev (not test), then repeated on clean split B
- **Luck check:** paired bootstrap, 1,000 resamples, 95% range; range including 0 = possible luck

### Results (AnCast)

| Split A | All | Minecraft | Non-Minecraft |
|---|---|---|---|
| Job 1 AMRBART | 86.9 | 92.6 | 66.8 |
| R1 Upsample | 86.7 | 92.5 | 66.9 |
| R2 Two-stage (dev winner) | 87.0 | 92.7 | 67.2 |
| R3 Combined | 86.7 | 92.5 | 66.4 |
| **Split B:** Job 1 AMRBART | 79.4 | 84.2 | 68.4 |
| **Split B:** R2 Two-stage | 79.0 | 84.0 | 67.7 |

### SMATCH gain on normal English vs. Job 1 AMRBART

| Run | Gain | 95% range |
|---|---|---|
| R1 Upsample (A) | −0.41 | −1.48 to +0.61 |
| R2 Two-stage (A) | +0.19 | −0.36 to +0.73 |
| R3 Combined (A) | −0.45 | −1.64 to +0.66 |
| R2, clean split B | −0.32 | −0.78 to +0.13 |

- **No fix helped**; every range includes 0
- 4× more normal-English examples (share 16% → 43%), no change → **data imbalance likely not the cause**
- More likely: normal English simply harder (longer, more varied, more complex)

---

## Summary

1. **Replication works:** 86.9 / 91.1 / 91.2 without the authors' code, vs. paper's 81.7 / 86.2 / 88.7
2. **Duplicates inflate scores:** ~half of a random test set in train; clean split 86.9 → 79.4 AnCast, all of the drop on Minecraft
3. **Real weakness = normal English:** ~26 AnCast below game chat
4. **General LLM no fix:** Qwen3-8B close on game chat, 9–10 worse on normal English
5. **More data no fix:** no significant gain → sentence difficulty, not data imbalance
6. **Tooling issues:** AMRBART decoder drops UMR attributes; AnCast can silently score one sentence

## Limitations

- One run per setting, no repeated seeds
- New random test sample (paper's not public)
- 66 AMR-overlap graphs not removed (for simplicity)
- Greedy decoding (LLM) vs. beam search (AMRBART)

## Files

| File | Purpose |
|---|---|
| `umr_tools.py` | Data reading, splits, scoring |
| `umr_tokenizer.py` | AMRBART tokenizer + UMR fixes + decoder |
| `train_amrbart.py` | AMRBART training and prediction |
| `llm_tools.py` | Qwen3 LoRA training and prediction |
| `job3_tools.py` | Upsampling, bootstrap test |
| `job1_*.ipynb`, `job2_*.ipynb`, `job3_*.ipynb` | Colab notebooks, one per job |
