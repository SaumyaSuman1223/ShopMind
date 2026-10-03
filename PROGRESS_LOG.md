# ShopMind — Progress Log

Running log of project work. Newest session at the bottom.

---

## Session 1 — 2026-10-03 — Raw dataset acquisition

**Goal:** read the architecture PDF, build the raw-datasets layer (`data/raw/`), acquire every dataset the
PDF calls for (or a justified substitute), and store everything on the external drive.

**Outcome:** 7 datasets (18 GB) acquired, checksum-verified, and stored on the external drive
`D:\ShopMind\data\raw\`. No local copies remain. A few sub-files are still missing because they are behind logins (see §6).

### 1. Architecture PDF analysis

Source: `ShopMind_Datasets_and_Catalogue_Architecture.pdf` (repo root; copy also at `D:\ShopMind\`).

ShopMind is an end-to-end e-commerce recommendation sandbox. A realistic shop website is driven by three
**independently trained** model systems:

| System | Training data (per PDF) | Model progression (per PDF) |
|---|---|---|
| Product recommendation | RetailRocket (behaviour) + Amazon Reviews 2023 (catalogue) | item-item / MF → LightGCN, SASRec, two-tower retrieval → learning-to-rank. Retrieval of 100 candidates → ranking → top 20 |
| Advertisement CTR / ranking | Criteo Display Advertising Challenge (+ simulated campaigns) | Logistic Regression → XGBoost/LightGBM → DeepFM → two-tower + DeepFM |
| Discount / coupon | Tianchi O2O (discount → redemption), Ponpare (user → coupon ranking) | XGBoost / neural → P(redeem) |
| Optional | Microsoft MIND (impression-based ranking) | — |

Other key points from the PDF:
- RetailRocket implicit-feedback weights: **view = 1, add-to-cart = 3, transaction = 5**.
- Amazon: use one or two categories (e.g. Electronics), build a 100k–500k product catalogue. Amazon is mainly the
  **catalogue/metadata source**; RetailRocket is the main behavioural training signal.
- Catalogue Option A: PostgreSQL `products` table (product_id, title, description, brand, category, subcategory,
  price, image, rating, stock) + OpenSearch/Elasticsearch. Option B: DummyJSON live API for early frontend prototyping only.
- Serving: catalogue → search → candidate retrieval → recommender → ranking → (organic recs | sponsored ads → CTR model) → discount engine → product page.
- Stack: Next.js + TypeScript + Tailwind, FastAPI, PostgreSQL, OpenSearch, pgvector, PyTorch + LightGBM + scikit-learn,
  MLflow, Redis, Docker, Vercel, Render/Railway/AWS. Start minimal: PostgreSQL + FastAPI + one model + simple frontend.
- Development order: **data → preprocessing → baseline models → offline evaluation → serving → catalogue → website → real events → experiments.**
- Core principle: keep training data, models, serving, catalogue, website and newly collected user events clearly separated.

### 2. Dataset survey (sizes, availability, alternatives)

Every URL below was probed directly (HTTP HEAD / Hugging Face API) before any download.

| Dataset | Size | Availability found | Alternatives identified |
|---|---|---|---|
| RetailRocket | 0.3 GB zipped / 0.95 GB unzipped | Provided by user (repo root) | — |
| Amazon Reviews 2023 — Electronics | meta 1.31 GB gz, reviews 6.47 GB gz, 0-core ratings 2.52 GB, 5-core ratings 0.90 GB | ✅ direct (mcauleylab.ucsd.edu, HF `McAuley-Lab/Amazon-Reviews-2023`) | Cell Phones (0.86 + 2.53 GB), Video Games, Appliances, Home & Kitchen, etc. |
| Criteo Kaggle DAC | 4.58 GB `.tar.gz` (≈12.6 GB unpacked) | ❌ original S3 link and `go.criteo.net` link return 404; ✅ **Figshare 5732310** works | HF `reczoo/Criteo_x1` (2.9 GB, pre-split 7:2:1); Avazu (`reczoo/Avazu_x1`, 0.6 GB); Criteo 1TB (`criteo/CriteoClickLogs`, 276 GB — unnecessary) |
| Tianchi O2O | ~140 MB zipped (est.) | ❌ official site requires an Alibaba Cloud login; ⚠️ partial GitHub mirrors | **dunnhumby The Complete Journey** (coupons + redemptions + campaigns); AmExpert 2019 coupon redemption |
| Ponpare | ~1 GB unpacked (est.) | ⚠️ Kaggle competition (API token + accept rules); GitHub mirror lacks visit log | — |
| MIND | small 84 MB, large 1.24 GB | ❌ Microsoft blob storage returns 409 (public access disabled); ⚠️ HF `yjw1029/MIND` gated (needs login); ✅ ungated HF mirror `huyva/MIND-small` | `reczoo/MIND_small_x1`, `chuong090703/MIND-small` |
| DummyJSON | < 1 MB | ✅ live API | — (not downloaded; frontend-only) |

Environment checks: Linux disk had ~854 GB free; no Kaggle credentials; Hugging Face CLI installed but not logged in; no sudo.

### 3. Decisions made (by user)

1. Amazon deferred at first, then **Electronics metadata + full (0-core) ratings** chosen; review text skipped.
2. Proceed with all other datasets using the best reachable source.
3. Delete zip files and other unneeded files.
4. Store all datasets on the external drive.
5. After verification, **delete local copies** — the external drive is the only copy.

Decisions made by Claude under that direction:
- Criteo: original Kaggle DAC archive from Figshare (not the pre-split reczoo version), so the raw layer stays unmodified.
- Tianchi: official site needs a login, so the core offline train + test files came from GitHub mirrors, and **dunnhumby
  Complete Journey** was added as a substitute/complement for discount-redemption modelling.
- MIND: MIND-small from the ungated `huyva/MIND-small` mirror (file sizes and row counts match the official MINDsmall release).
- Amazon "full ratings": interpreted as `benchmark/0core/rating_only` (all ratings, no k-core filter); 5-core can be derived later.

### 4. What was done, step by step

1. **RetailRocket**: extracted `events.csv.zip`, `item_properties_part1.csv.zip` and `item_properties_part2.csv.zip` and moved
   `category_tree.csv` into `data/raw/retailrocket/`. Row counts verified against the official release. Zips deleted.
2. **`.gitignore`** created: `data/raw/`, `data/interim/`, `data/processed/` (datasets are never committed).
3. **Criteo**: downloaded `dac.tar.gz` from Figshare and verified its md5 against Figshare's published value
   (`df9b1b3766d9ff91d5ca3eb3d23bed27`). Extracted `train.txt`, `test.txt` and `readme.txt`, then deleted the archive.
4. **Tianchi O2O**: `ccf_offline_stage1_train.rar` came from GitHub `zodiac-dy/o2o-coupon-forecast` and was extracted with RARLAB
   `unrar` (temporary binary); `ccf_offline_stage1_test_revised.csv` came from GitHub `llv22/O2O_coupon`. The .rar was deleted.
5. **Ponpare**: 7 CSVs from GitHub `Sulagna-Dutta-Roy/Coupon-Prediction`.
6. **MIND-small**: train/dev `behaviors.tsv` + `news.tsv` from HF `huyva/MIND-small`.
7. **dunnhumby Complete Journey**: `.rda`/`.rds` files from GitHub `bradleyboehmke/completejourney` (2017 release, full
   transactions + promotions), converted to CSV. The small tables used `pyreadr` (temporary venv). `promotions` (20.9M rows) ran out of memory in
   Python on this 7 GB-RAM machine, so it was converted with a temporary micromamba R + `data.table::fwrite`. Source R files were deleted.
8. **External drive**: identified as **D: "My Passport"** (NTFS, 931 GB). User mounted it in WSL with
   `sudo mount -t drvfs D: /mnt/d -o metadata`.
9. **SHA256SUMS** written in every dataset folder, then everything was copied to `D:\ShopMind\data\raw\` and verified on the drive:
   - small datasets: `cp` + `sha256sum -c` on the drive;
   - large files: Windows `robocopy` + Windows-side hashing (`Get-FileHash` / `certutil`), compared with the local SHA256SUMS.
   - Before deleting local data, all 34 local files were compared against the drive copy (0 mismatches).
10. **Local copies deleted** (`data/raw/` removed). The PDF and `data/README.md` were copied to the drive.
11. **Amazon Electronics**: downloaded to local staging and verified. The ratings sha256 matches Hugging Face's published LFS hash, and the metadata passed
    `gzip -t` with a line count. Both were robocopied to the drive, verified with `certutil` hashes, and the local staging was deleted.
12. Temporary tooling (Python venv, R environment, unrar, intermediate files) was deleted from the session scratchpad.
13. **`data/README.md`** written: dataset catalogue, sources, row counts, known gaps, drive/mount notes.

### 5. Final inventory — `D:\ShopMind\data\raw\` (18 GB total)

| Folder | Size | Files | Rows (data rows, excl. header) |
|---|---|---|---|
| `amazon_reviews_2023/` | 3.6 GB | `meta_Electronics.jsonl.gz`, `Electronics.ratings_0core.csv` | 1,610,012 products; 43,365,426 ratings (`user_id,parent_asin,rating,timestamp`) |
| `criteo/` | 12 GB | `train.txt`, `test.txt`, `readme.txt` | train 45,840,617 (label + I1–I13 + C1–C26, TSV, no header); test 6,042,135 (unlabelled) |
| `retailrocket/` | 942 MB | `events.csv`, `item_properties_part1.csv`, `item_properties_part2.csv`, `category_tree.csv` | events 2,756,101 (view 2,664,312 / addtocart 69,332 / transaction 22,457); properties 10,999,999 + 9,275,903; categories 1,669 |
| `dunnhumby_completejourney/` | 486 MB | `transactions`, `promotions`, `products`, `coupons`, `coupon_redemptions`, `campaigns`, `campaign_descriptions`, `demographics` (.csv) | transactions 1,469,307; promotions 20,940,529 |
| `mind/` | 200 MB | `train/` and `dev/` `behaviors.tsv` + `news.tsv` | train 156,965 impressions / 51,282 news; dev 73,152 / 42,416 |
| `tianchi_o2o/` | 73 MB | `ccf_offline_stage1_train.csv`, `ccf_offline_stage1_test_revised.csv` | 1,754,884; 113,640 |
| `ponpare/` | 36 MB | `user_list`, `coupon_list_train/test`, `coupon_detail_train`, `coupon_area_train/test`, `prefecture_locations` (.csv) | users 22,873; coupons 19,413; purchases 168,996 |

Also on the drive: `D:\ShopMind\ShopMind_Datasets_and_Catalogue_Architecture.pdf`, `D:\ShopMind\data\README.md`.
Every dataset folder has a `SHA256SUMS`. Checksums recorded during this session:

```
# retailrocket
94e865eb0a3d48cbbfe3b79079018dd92509315c88f5fd8d00d0b4b5af434f5b  category_tree.csv
3745aa83238b1e6d44d8fda209807899f420084398f94ddf745f3cbcfecbf9e7  events.csv
30aad5aeca58b2dc27dcc73e1708565f5818e45adb3eb57401f91e87355b0b81  item_properties_part1.csv
d5e7d1a91dc40f522aeb596b267e6c87d8aed689a7192d12369cfb165eb987e5  item_properties_part2.csv
# criteo
df4f65430e31d8f9234b99bb7e8eb048899189d6bbdd59acfc26def08f0bd90b  readme.txt
7ff021b11faa3b7468fecbfbf987e5ebe4563f7d4c4db462c7d2ff55a08e1db6  test.txt
f09ccdc7e76403030cd56455b8c16fde6a2ab73a9e3736a1563ffb0cd02d68fb  train.txt
# dunnhumby_completejourney (large files)
85d190f38dde9fdf2c365bbd90c4c33f7e67b5cbad65689e7f0a203d63822ebc  promotions.csv
9b2426b04909c8080cde21712794c0f5071a9a92fd9a99d0d80b50ea92933d6b  transactions.csv
# amazon_reviews_2023
a4a196a1c8e443e0942d8a5c79b5a5d2d68e29d483f2badec1126690b4b2790d  meta_Electronics.jsonl.gz
a2870fcfa23755418254a4329951f2916f068f7a078bbade67a09e85b818aa94  Electronics.ratings_0core.csv
```

### 6. Known gaps (not yet acquired)

| Missing | Why | How to get it |
|---|---|---|
| Tianchi `ccf_online_stage1_train.csv` (~11.4M rows online behaviour) | Only on tianchi.aliyun.com | Alibaba Cloud login, competition 231593 |
| Ponpare `coupon_visit_train.csv` (browsing log) | Kaggle competition data | Kaggle API token in `~/.kaggle/kaggle.json` + accept competition rules |
| MIND-large, MIND entity embeddings | Gated on Hugging Face | `hf auth login`, then `yjw1029/MIND` |
| Amazon review text (Electronics, 6.47 GB) | Skipped by choice | Direct download from mcauleylab.ucsd.edu if needed later |
| DummyJSON | Not needed yet (frontend prototyping only) | Live API |

### 7. Issues hit and how they were solved

| Issue | Fix |
|---|---|
| Criteo original links (S3, `go.criteo.net`) return 404 | Used the Figshare mirror; verified against Figshare's md5 |
| Microsoft MIND blob storage returns 409 (public access disabled) | Ungated HF mirror `huyva/MIND-small` |
| No `unrar` / `R` installed, no sudo | Temporary RARLAB `unrar` binary + micromamba R env in the scratchpad (deleted afterwards) |
| `pyreadr` killed (out of memory) converting 20.9M-row `promotions.rds` — machine has ~7 GB RAM, with other projects' jobs using ~3 GB | Converted with R `readRDS` + `data.table::fwrite` |
| `/mnt/d` existed but was **not** the drive (empty dir on the Linux disk) | User mounted D: with `sudo mount -t drvfs D: /mnt/d -o metadata` |
| `rsync`/`cp`/`sha256sum` on the drive failed with **"Cannot allocate memory"** for multi-GB files (WSL drvfs under memory pressure) | Copy with Windows `robocopy` from `\\wsl.localhost\Ubuntu\...`; hash on the Windows side |
| WSL→Windows interop timeout (`UtilAcceptVsock: accept4 failed 110`) when calling PowerShell | Retried; used `certutil -hashfile` via `cmd.exe` |

### 8. Operational notes (external drive)

```bash
# after plugging in the drive or restarting WSL
sudo mount -t drvfs D: /mnt/d -o metadata

# before unplugging
sudo umount /mnt/d
powershell.exe -NoProfile -Command '(New-Object -ComObject Shell.Application).Namespace(17).ParseName("D:").InvokeVerb("Eject")'
```

- Large-file copies to/from the drive: use `robocopy`, not `cp`/`rsync`.
- Verify large files on the drive with Windows tools (`certutil -hashfile <file> SHA256` or `Get-FileHash`).
- Reading through drvfs is slow; for training, copy the needed files to local `data/raw/` (git-ignored) first.

### 9. Design notes and open questions for next steps

- **RetailRocket cannot be joined to Amazon.** RetailRocket item IDs and property values are hashed/anonymised. Plan:
  train the *serving* product recommender on Amazon Electronics ratings (IDs match the catalogue) and use RetailRocket
  for event-type weighting (view 1 / cart 3 / purchase 5), session models (SASRec) and offline benchmarking.
- Amazon metadata has many items without a price. Catalogue filtering (price + image + min ratings) should bring 1.6M items
  down to the PDF's 100k–500k target.
- Tianchi offline data (`Discount_rate` such as `150:20` = spend 150 save 20, or a fraction) drives the discount→redemption
  model. dunnhumby adds US retail coupon redemptions and campaign exposure.

### 10. Repo state at end of session

```
ShopMind/
├── .gitignore            # data/raw/, data/interim/, data/processed/
├── PROGRESS_LOG.md       # this file
├── README.md
├── ShopMind_Datasets_and_Catalogue_Architecture.pdf
└── data/
    └── README.md         # dataset catalogue, sources, row counts, gaps
```

Nothing committed yet (only the initial commit `808f6be` exists).

### Next steps (per PDF development order)

1. Preprocessing: RetailRocket weighted interactions + temporal train/val/test split; Amazon Electronics catalogue
   filtering → `products` table schema; Criteo feature encoding; Tianchi discount parsing + redemption label.
2. Baseline models + offline evaluation (Recall@K / NDCG@K for recs, AUC/logloss for CTR and redemption).
3. Optionally fill the gaps in §6 (Kaggle token for Ponpare visits, HF login for MIND-large).

### Sources

- Amazon Reviews 2023: https://amazon-reviews-2023.github.io/ · https://huggingface.co/datasets/McAuley-Lab/Amazon-Reviews-2023
- Criteo DAC (Figshare): https://figshare.com/articles/dataset/Kaggle_Display_Advertising_Challenge_dataset/5732310
- Criteo 1TB / alternatives: https://ailab.criteo.com/download-criteo-1tb-click-logs-dataset/ · https://huggingface.co/datasets/reczoo/Criteo_x1
- Tianchi O2O: https://tianchi.aliyun.com/competition/entrance/231593/information · https://github.com/zodiac-dy/o2o-coupon-forecast · https://github.com/llv22/O2O_coupon
- Ponpare: https://www.kaggle.com/c/coupon-purchase-prediction · https://github.com/Sulagna-Dutta-Roy/Coupon-Prediction
- MIND: https://huggingface.co/datasets/huyva/MIND-small · https://huggingface.co/datasets/yjw1029/MIND
- dunnhumby Complete Journey: https://github.com/bradleyboehmke/completejourney
