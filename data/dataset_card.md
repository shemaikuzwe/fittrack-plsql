# Dataset Card — FitTrack Gym Operations


## 1. Identity

| Field | Value |
|---|---|
| Dataset title | Indian Gym Engagement & Retention Dataset |
| Publisher | Kaggle user `diyashrivastava12` |
| Source link | https://www.kaggle.com/datasets/diyashrivastava12/indian-gym-engagement-and-retention-dataset |
| Licence | **CC0 1.0 — Public Domain** (no attribution required; credited here anyway) |
| Format | CSV |
| Files | 5 CSV files, 27.3 MB |
| Date collected | **2026-11-22** |
| Real or synthetic | **Fully synthetic.** Publisher states no real individuals, gyms or transactions are represented. Value distributions are modelled on Indian fitness-industry patterns. |
| Integrity | `sha256sum -c data/raw/checksums.txt` must pass |

## 2. What one record represents

Each file has its own grain. Getting this wrong is the most common Phase II mistake, so it is stated explicitly.

| File | Rows | Columns | One record represents |
|---|---|---|---|
| `members_master.csv` | 17,500 | 17 | One **gym member** (demographics, home branch, current tier) |
| `subscription_renewal_records.csv` | 45,764 | 13 | One **billing cycle** of one member (a renewal, upgrade or lapse) |
| `gym_checkin_workout_logs.csv` | 108,744 | 8 | One **gym visit** by one member (check-in, optional check-out) |
| `app_engagement_events.csv` | 97,988 | 7 | One in-app interaction — **unused, out of scope** |
| `streak_history_episodes.csv` | 40,202 | 8 | One workout-streak episode — **unused, out of scope** |

The three used files join on `member_id`. There are **0 orphan foreign keys** across all three.

## 3. Scope decisions

| Decision | Reason |
|---|---|
| Use `members_master`, `subscription_renewal_records`, `gym_checkin_workout_logs` | These carry members, plans/payments and attendance — the three pillars of FitTrack. |
| Retain but do not use `app_engagement_events.csv`, `streak_history_episodes.csv` | The project proposal puts the mobile app **out of scope**. Files are kept so `data/raw/` stays byte-identical to the download; see `normalization.md` §7. |
| No plan catalogue | The source has no price list. See §6 gap 1. |
| No trainer attributes | The source has trainer IDs only. See §6 gap 2. |

## 4. Quality report


### 4.1 `members_master.csv` — 17,500 rows

| Issue | Count |
|---|---|
| Blank `date_of_birth` | **540** (3.1%) |
| Near-duplicate members (same `full_name` + `date_of_birth`) | **848** extra rows in **815** groups |
| Distinct `full_name` values for 17,500 members | **4,132** — most repeated: `Sameer Pawar` **17×** |
| Raw `city` strings vs real cities | **81 → 29** after trimming whitespace → **27** after normalising spellings |
| City spelling variants | `Bengaluru`/`Bangalore`, `Gurugram`/`Gurgaon`, plus leading/trailing spaces |
| Branches split across two spellings of one city | **12 of 40** (e.g. `BR-033` holds 6 raw strings for 2 cities) |
| Cities whose `city_tier` is not unique | **28 of 29** (e.g. Bangalore: 765×Tier1, 10×Tier3, 3×Tier2) |
| `join_date` in `dd-mm-yyyy` | **9,663** |
| `join_date` in `yyyy/mm/dd` | **7,837** |
| `date_of_birth` in `dd-mm-yyyy` / `yyyy/mm/dd` / blank | **9,253 / 7,707 / 540** |
| Booleans stored as text `"True"`/`"False"` | `has_personal_trainer` (all 17,500 rows) |
| Free-text categories needing a lookup | `gender` (3), `occupation` (7), `marital_status` (3), `referral_source` (6), `preferred_workout_time` (5), `primary_goal` (6), `signup_channel` (3) |

### 4.2 `subscription_renewal_records.csv` — 45,764 rows

| Issue | Count |
|---|---|
| Duplicate rows — same `renewal_id` repeated (UPI retries) | **676** (45,088 distinct IDs of 45,764 rows) |
| `renewal_id` is therefore **not a unique key** in the raw file | 676 collisions |
| Renewed **after** expiry (`days_before_expiry_renewed` < 0) | **12,216** (26.7%); range −29 … +41 |
| `billing_cycle_end` − `billing_cycle_start` ≠ plan duration | **931** (e.g. a 16-day "Monthly" cycle) |
| Overlapping cycles for the same member | **676** (the same retry duplicates) |
| `discount_applied` fully derivable from `discount_pct` | **0** inconsistencies in 45,764 rows |
| `amount_paid_inr` stored as integer text | 4,611 distinct values |
| No plan catalogue — price varies for the same plan | **941** distinct amounts across 18,042 `Monthly` renewals at 0% discount (₹420 – ₹10,630) |
| No payment date column exists | — (see §6 gap 3) |
| Enum values to constrain | `renewal_status` (5), `payment_method` (5), `renewal_channel` (3) |

### 4.3 `gym_checkin_workout_logs.csv` — 108,744 rows

| Issue | Count |
|---|---|
| Blank `checkout_datetime` (member did not check out) | **13,144** (12.1%) |
| Blank `trainer_id` | **96,099** (88.4%) — populated **only** on `Personal Training Session` rows |
| `duration_minutes` ≠ `checkout − checkin` | **86,045 of 95,600** (90.0%) — the two measure different things |
| Repeated `checkin_datetime` values (different members) | **101** — not duplicate visits |
| Members with no check-in at all | **1,763** (10.1%) |
| Check-ins at a branch other than the member's home branch | **5,223** (4.8%) |
| `trainer_id` values with no attribute record anywhere | **50 IDs** |
| `workout_type` free text (4 values) | Cardio, Strength Training, Group Class, Personal Training Session |

### 4.4 Cross-file

| Check | Result |
|---|---|
| Renewals referencing unknown members | **0** |
| Check-ins referencing unknown members | **0** |
| Check-ins before the member's `join_date` | **0** |
| Members with zero renewals | **0** |
| Max renewals held by one member | **22** |

## 5. Anonymisation

The source is **fully synthetic** — the publisher states that no real individuals, gyms or transactions are represented. There is no real personal data to remove, so `data/raw/` was **not modified**, which also satisfies the rule "never edit this file".

Precautions taken in the design:

- `member_id` is the only member identifier used in the logical model and in all reports.
- `full_name` is retained (it is synthetic and the gym needs it) but is never used as a key.
- No phone numbers, emails, national IDs or addresses exist in the source.

## 6. Reproduce

```bash
mkdir -p data/raw && cd data/raw
curl -L -A "Mozilla/5.0" -o src.zip \
  "https://www.kaggle.com/api/v1/datasets/download/diyashrivastava12/indian-gym-engagement-and-retention-dataset"
unzip src.zip && rm src.zip
sha256sum -c checksums.txt
```
