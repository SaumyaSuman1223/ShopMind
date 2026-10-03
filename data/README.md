# ShopMind raw datasets

Raw data lives **only on the external drive** ("My Passport", `D:`) at `D:\ShopMind\data\raw\<dataset>\`
— `/mnt/d/ShopMind/data/raw` in WSL. Mount it after plugging in / restarting WSL:
`sudo mount -t drvfs D: /mnt/d -o metadata`. A local `data/raw/` (git-ignored) is only for working copies.

Each folder has a `SHA256SUMS`. Verify large files from Windows (`Get-FileHash -Algorithm SHA256 <file>`):
WSL's drvfs mount can fail with "Cannot allocate memory" when reading or writing multi-GB files under memory pressure.
For the same reason, copy large files to/from the drive with `robocopy` rather than `cp`/`rsync`.

| Folder | Role in ShopMind | Source | Size | Status |
|---|---|---|---|---|
| `retailrocket/` | Product recommendation (behavioural events) | Kaggle: retailrocket/ecommerce-dataset | 0.95 GB | Complete |
| `criteo/` | Ad CTR model | Criteo Kaggle Display Advertising Challenge (2014), via [Figshare 5732310](https://figshare.com/articles/dataset/Kaggle_Display_Advertising_Challenge_dataset/5732310) (md5 `df9b1b3766d9ff91d5ca3eb3d23bed27`) | 12.6 GB | Complete |
| `tianchi_o2o/` | Discount → redemption model | Tianchi O2O Coupon Usage Forecast (competition 231593), via GitHub mirrors `zodiac-dy/o2o-coupon-forecast`, `llv22/O2O_coupon` | 76 MB | Offline train + test only |
| `ponpare/` | User → coupon recommendation | Kaggle: coupon-purchase-prediction (Recruit Ponpare), via GitHub mirror `Sulagna-Dutta-Roy/Coupon-Prediction` | 37 MB | Missing `coupon_visit_train.csv` |
| `mind/` | Optional: impression-based ranking | Microsoft MIND-small, via Hugging Face `huyva/MIND-small` | 0.2 GB | train/dev behaviors + news only |
| `dunnhumby_completejourney/` | Discount/coupon redemption (substitute/complement for Tianchi) | dunnhumby The Complete Journey (2017 release), via R package `bradleyboehmke/completejourney`, converted .rda/.rds → CSV | 0.49 GB | Complete |
| `amazon_reviews_2023/` | Product catalogue + catalogue-side recommender (Electronics) | [Amazon Reviews 2023](https://amazon-reviews-2023.github.io/), McAuley Lab: `raw/meta_categories/meta_Electronics.jsonl.gz` (mcauleylab.ucsd.edu) + `benchmark/0core/rating_only/Electronics.csv` (HF `McAuley-Lab/Amazon-Reviews-2023`, saved as `Electronics.ratings_0core.csv`) | 3.8 GB | Metadata + all ratings; no review text |

## Row counts (data rows, excluding header)

- **retailrocket**: events 2,756,101 (view 2,664,312 / addtocart 69,332 / transaction 22,457);
  item_properties part1 10,999,999 + part2 9,275,903; category_tree 1,669
- **criteo**: `train.txt` 45,840,617 labelled impressions (label, I1–I13, C1–C26, tab-separated, no header); `test.txt` 6,042,135 unlabelled
- **tianchi_o2o**: `ccf_offline_stage1_train.csv` 1,754,884; `ccf_offline_stage1_test_revised.csv` 113,640
- **ponpare**: user_list 22,873; coupon_list_train 19,413; coupon_detail_train (purchases) 168,996;
  coupon_area_train 138,185; coupon_list_test 310; coupon_area_test 2,165; prefecture_locations 47
- **mind**: train behaviors 156,965 / news 51,282; dev behaviors 73,152 / news 42,416
- **amazon_reviews_2023**: `meta_Electronics.jsonl.gz` 1,610,012 products (one JSON object per line);
  `Electronics.ratings_0core.csv` 43,365,426 ratings (`user_id,parent_asin,rating,timestamp`, ms epoch).
  Join on `parent_asin`. RetailRocket item IDs are anonymised and cannot be joined to Amazon products.
- **dunnhumby_completejourney**: transactions 1,469,307; promotions 20,940,529; products, coupons,
  coupon_redemptions, campaigns, campaign_descriptions, demographics

## Known gaps

- **Tianchi `ccf_online_stage1_train.csv`** (online behaviour, ~11.4M rows): only on tianchi.aliyun.com (Alibaba Cloud login).
- **Ponpare `coupon_visit_train.csv`** (browsing log): needs a Kaggle API token + accepting the competition rules.
- **MIND-large** and MIND entity embeddings: `yjw1029/MIND` on Hugging Face (requires `hf auth login`).
