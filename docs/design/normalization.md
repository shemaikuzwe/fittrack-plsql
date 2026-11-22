# Normalisation — UNF → 1NF → 2NF → 3NF

Phase II deliverable. Every row quoted below is copied from `data/raw/`; nothing is invented.
Counts come from `data/dataset_card.md`.

---

## 1. Source and grains

Three related CSV files, joined on `member_id`:

| File | Rows | Grain |
|---|---|---|
| `members_master.csv` | 17,500 | one member |
| `subscription_renewal_records.csv` | 45,764 | one billing cycle |
| `gym_checkin_workout_logs.csv` | 108,744 | one visit |

The two unused files (`app_engagement_events.csv`, `streak_history_episodes.csv`) are out of scope — see §8.

---

## 2. UNF — unnormalised form

A gym system exporting "renewals with member details" produces the record below: member attributes, a **list of renewals**, and a **list of visits** in one row. That is our UNF relation.

### 2.1 Member `MBR-005003` — 17 attributes in one record

| Field | Value |
|---|---|
| member_id | MBR-005003 |
| full_name | Sneha Desai |
| gender | Female |
| date_of_birth | 25-12-1987 |
| city | Mumbai |
| city_tier | Tier1 |
| state | Maharashtra |
| occupation | Freelancer |
| marital_status | Single |
| gym_branch_id | BR-039 |
| join_date | 2021/01/20 |
| membership_type | Premium |
| referral_source | Walk-in |
| preferred_workout_time | Morning |
| has_personal_trainer | False |
| primary_goal | Weight Loss |
| signup_channel | Website |

### 2.2 … plus a repeating group of 4 renewals

| Field | REN-012936 | REN-012937 | REN-012937 (repeat) | REN-012938 |
|---|---|---|---|---|
| plan_type | Monthly | Monthly | Monthly | Monthly |
| start | 2021-01-20 | 2021-04-26 | 2021-04-26 | 2021-05-27 |
| end | 2021-02-09 | 2021-05-26 | 2021-05-26 | 2021-06-26 |
| amount | 3570 | 4180 | 4180 | 8270 |
| method | UPI | UPI | UPI | UPI |
| discount_applied | False | True | True | False |
| discount_pct | 0 | 5 | 5 | 0 |
| status | Lapsed | Renewed | Renewed | Renewed |
| days_before_expiry | 10 | -10 | -10 | -6 |
| channel | App | Phone Call from Staff | Phone Call from Staff | App |
| corporate | False | False | False | False |

### 2.3 … plus a repeating group of 14 visits (3 shown)

| Field | CHK-002234 | CHK-002962 | CHK-004534 |
|---|---|---|---|
| branch_id | BR-039 | BR-039 | BR-039 |
| checkin | 2021-09-19 04:16 | 2021-11-02 16:23 | 2022-01-11 20:05 |
| checkout | 2021-09-19 05:42 | 2021-11-02 17:28 | 2022-01-11 20:47 |
| workout_type | Group Class | Cardio | Cardio |
| trainer_id | *(blank)* | *(blank)* | *(blank)* |
| duration_minutes | 81 | 67 | 43 |

**Problems:** two repeating groups; `REN-012937` appears twice; `REN-012936` is a 20-day "Monthly" cycle; `discount_applied` duplicates `discount_pct`; `city`, `city_tier` and `state` repeat identically on all 45,764 renewal rows.

---

## 3. 1NF — remove repeating groups

> **Dependency removed — repeating group:** one member record held a list of renewals and a list of visits; each list becomes its own table with one row per event, and every column holds a single value.

Keys after 1NF: `member_id`, `renewal_id`, `checkin_id`.

### 3.1 `SUBSCRIPTION` after 1NF (4 real rows, all for `MBR-005003`)

| renewal_id | plan_type | start | end | amount |
|---|---|---|---|---|
| REN-012936 | Monthly | 2021-01-20 | 2021-02-09 | 3570 |
| REN-012937 | Monthly | 2021-04-26 | 2021-05-26 | 4180 |
| REN-012937 | Monthly | 2021-04-26 | 2021-05-26 | 4180 |
| REN-012938 | Monthly | 2021-05-27 | 2021-06-26 | 8270 |

**Uniqueness defect found:** `renewal_id` is **not unique** — 676 rows across the file repeat an existing ID (UPI retries). 1NF alone does not fix this; it is resolved at 2NF by giving the payment its own key.

### 3.2 `ATTENDANCE` after 1NF (3 of 14 real visits, all by `MBR-005003`)

| checkin_id | branch_id | checkin | checkout | minutes |
|---|---|---|---|---|
| CHK-002234 | BR-039 | 2021-09-19 04:16 | 2021-09-19 05:42 | 81 |
| CHK-002962 | BR-039 | 2021-11-02 16:23 | 2021-11-02 17:28 | 67 |
| CHK-004534 | BR-039 | 2022-01-11 20:05 | 2022-01-11 20:47 | 43 |

---

## 4. 2NF — remove partial dependencies

> **Dependency removed — partial dependency:** with the natural key `(member_id, billing_cycle_start)`, `full_name`, `gender`, `date_of_birth`, `occupation`, `marital_status`, `join_date`, `referral_source`, `preferred_workout_time`, `has_personal_trainer`, `primary_goal` and `signup_channel` depend on `member_id` alone, so they move to `MEMBER`.

### 4.1 The joined export — member attributes repeat on every renewal

| Field | MBR-005003 @ 2021-01-20 | MBR-005003 @ 2021-04-26 | MBR-005003 @ 2021-05-27 | MBR-009827 @ 2024-07-27 |
|---|---|---|---|---|
| full_name | Sneha Desai | Sneha Desai | Sneha Desai | Priya Iyengar |
| gender | Female | Female | Female | Female |
| date_of_birth | 25-12-1987 | 25-12-1987 | 25-12-1987 | 21-07-1994 |
| occupation | Freelancer | Freelancer | Freelancer | Salaried-Non IT |
| marital_status | Single | Single | Single | Single |
| join_date | 2021/01/20 | 2021/01/20 | 2021/01/20 | 2024/07/27 |
| membership_type | Premium | Premium | Premium | Basic |
| referral_source | Walk-in | Walk-in | Walk-in | Instagram Ad |
| amount_paid_inr | 3570 | 4180 | 8270 | 5760 |

The member block is identical for all rows of `MBR-005003` and different for `MBR-009827` → it is a partial dependency, not a property of the renewal. It moves to **MEMBER**.

The same applies to `ATTENDANCE`, keyed `(member_id, checkin_datetime)`.

### 4.2 Second partial dependency — payments

A renewal can carry more than one payment attempt (the 676 retry duplicates). With key `(renewal_id, payment_seq)`:

- `plan_type`, `start_date`, `end_date`, `status`, `days_before_expiry`, `renewal_channel`, `corporate_tie_up` depend on `renewal_id` only → **SUBSCRIPTION**
- `amount_paid_inr`, `payment_method`, `discount_pct` depend on the payment attempt → **PAYMENT**

| renewal_id | payment_seq | amount | method | discount_pct |
|---|---|---|---|---|
| REN-012936 | 1 | 3570 | UPI | 0 |
| REN-012937 | 1 | 4180 | UPI | 5 |
| REN-012937 | 2 | 4180 | UPI | 5 |
| REN-012938 | 1 | 8270 | UPI | 0 |

This also fixes the `renewal_id` uniqueness defect: `PAYMENT` gets its own key, so a retry is a second payment row instead of a duplicated subscription.

---

## 5. 3NF — remove transitive dependencies

### 5.1 Branch: member → branch → city

> **Dependency removed — transitive dependency:** `city`, `city_tier` and `state` depend on `gym_branch_id`, not on the member, so they move to `BRANCH`.

Real evidence that city is a property of the branch, not the member — branch `BR-033` holds **six** raw spellings of two cities:

| gym_branch_id | raw `city` values found |
|---|---|
| BR-033 | `' Bangalore'`, `'Bengaluru '`, `'Bengaluru'`, `'Bangalore '`, `' Bengaluru'`, `'Bangalore'` |
| BR-010 | `'Chennai '`, `'Chennai'`, `' Chennai'` |

40 branches, 29 trimmed city strings, 27 real cities. **12 of 40 branches** are split this way.

| branch_id | city | city_tier | state |
|---|---|---|---|
| BR-033 | Bangalore | Tier1 | Karnataka |
| BR-039 | Mumbai | Tier1 | Maharashtra |
| BR-010 | Chennai | Tier1 | Tamil Nadu |

### 5.2 Plan: renewal → plan → duration

> **Dependency removed — transitive dependency:** `duration_days` depends on `plan_type` (Monthly = 30, Quarterly = 91, Half-Yearly = 182, Annual = 365), not on the individual renewal, so it moves to `MEMBERSHIP_PLANS`.

| plan_id | billing_cycle | duration_days |
|---|---|---|
| P-M | Monthly | 30 |
| P-Q | Quarterly | 91 |
| P-H | Half-Yearly | 182 |
| P-A | Annual | 365 |

**Kept in SUBSCRIPTION:** `end_date`, because **931** cycles do not match the plan duration (e.g. `REN-012936` is a 20-day Monthly cycle). It is therefore *not* functionally determined by `start_date + duration_days`, and must be stored.

### 5.3 Trainer: visit → trainer → name/specialty

> **Dependency removed — transitive dependency:** a trainer's name and specialty depend on `trainer_id`, not on the visit, so they move to `TRAINERS`.

`TRAINERS` holds 50 seeded reference rows. The source has only the 50 IDs — no names or specialties — which is recorded as a gap in the dataset card.

### 5.4 Dropped, not moved: `discount_applied`

`discount_applied` is `True` exactly when `discount_pct > 0` — **0** exceptions in 45,764 rows. It is functionally determined by another attribute in the same row, so it is dropped rather than moved to a table.

---

## 6. Derived columns — kept or dropped, with reasons

| Column | Derived from | Decision |
|---|---|---|
| `discount_applied` | `discount_pct > 0` | **Dropped** — 0 inconsistencies |
| `end_date` | `start_date + duration_days` | **Kept** — 931 cycles disagree, so it is independent |
| `duration_minutes` | `checkout − checkin` | **Kept** — 86,045 of 95,600 disagree (90%); the two measure different things |
| `days_before_expiry_renewed` | previous cycle's `end_date` | **Kept** — cross-row derivation, used by `prc_expiry_alerts`; 12,216 rows are negative (renewed late) |
| `member_status` (Active/Expired) | `end_date` vs today | **Not stored** — computed by `fn_is_active` in Phase IV |
| `list_price` | — | **Not derivable** — 941 distinct amounts for the same Monthly plan (₹420–₹10,630) |

---

## 7. Field mapping (task 5)

Every raw field is mapped, or dropped with a reason. "Conversion" is the Phase IV cleaning checklist.

### 7.1 `members_master.csv` → 17 fields

| Raw field | Target | Conversion / cleaning |
|---|---|---|
| `member_id` | MEMBER.member_id | none — already unique (17,500/17,500) |
| `full_name` | MEMBER.full_name | none |
| `gender` | MEMBER.gender | text → enum (Male 10,747 / Female 6,371 / Other 382) |
| `date_of_birth` | MEMBER.date_of_birth | **text → DATE**, two formats (`dd-mm-yyyy` 9,253, `yyyy/mm/dd` 7,707); 540 blank → NULL |
| `city` | BRANCH.city | trim whitespace, normalise Bengaluru/Bangalore and Gurugram/Gurgaon |
| `city_tier` | BRANCH.city_tier | text → enum; **28 of 29 cities conflict** — pick dominant value |
| `state` | BRANCH.state | trim; 17 states |
| `occupation` | MEMBER.occupation | text → enum (7) |
| `marital_status` | MEMBER.marital_status | text → enum (3) |
| `gym_branch_id` | MEMBER.branch_id | text → FK to BRANCH |
| `join_date` | MEMBER.join_date | **text → DATE**, two formats (`dd-mm-yyyy` 9,663, `yyyy/mm/dd` 7,837) |
| `membership_type` | MEMBER.membership_tier | text → enum (Basic/Standard/Premium/PT-Combo) |
| `referral_source` | MEMBER.referral_source | text → enum (6) |
| `preferred_workout_time` | MEMBER.preferred_workout_time | text → enum (5) |
| `has_personal_trainer` | MEMBER.has_personal_trainer | **`"True"`/`"False"` text → Y/N** |
| `primary_goal` | MEMBER.primary_goal | text → enum (6) |
| `signup_channel` | MEMBER.signup_channel | text → enum (3) |

### 7.2 `subscription_renewal_records.csv` → 13 fields

| Raw field | Target | Conversion / cleaning |
|---|---|---|
| `renewal_id` | SUBSCRIPTION.subscription_id | text; **676 duplicate IDs → dedupe** |
| `member_id` | SUBSCRIPTION.member_id | text → FK |
| `plan_type` | SUBSCRIPTION.plan_id (FK) | text → lookup into MEMBERSHIP_PLANS |
| `billing_cycle_start` | SUBSCRIPTION.start_date | **text → DATE** |
| `billing_cycle_end` | SUBSCRIPTION.end_date | **text → DATE**; 931 cycles ≠ plan duration |
| `amount_paid_inr` | PAYMENT.amount | **text → NUMBER** |
| `payment_method` | PAYMENT.method | text → enum (UPI 25,347 / Debit Card 7,244 / Credit Card 7,203 / Net Banking 3,677 / Cash 2,293) |
| `discount_applied` | — | **DROPPED** — derivable from `discount_pct > 0` |
| `discount_pct` | PAYMENT.discount_pct | **text → NUMBER**; 0–40% |
| `renewal_status` | SUBSCRIPTION.status | text → enum (Renewed 34,855 / Lapsed 4,523 / Auto-Renew Failed 2,757 / Upgraded 1,840 / Downgraded 1,789) |
| `days_before_expiry_renewed` | SUBSCRIPTION.days_before_expiry | **text → NUMBER**; 12,216 negative (renewed after expiry) |
| `renewal_channel` | SUBSCRIPTION.renewal_channel | text → enum (App / Front Desk / Phone Call from Staff) |
| `corporate_tie_up_flag` | SUBSCRIPTION.corporate_tie_up | **`"True"`/`"False"` text → Y/N** |

### 7.3 `gym_checkin_workout_logs.csv` → 8 fields

| Raw field | Target | Conversion / cleaning |
|---|---|---|
| `checkin_id` | ATTENDANCE.checkin_id | text; unique (108,744/108,744) |
| `member_id` | ATTENDANCE.member_id | text → FK |
| `branch_id` | ATTENDANCE.branch_id | text → FK; differs from home branch on 5,223 rows (valid) |
| `checkin_datetime` | ATTENDANCE.checkin_time | **text → TIMESTAMP** |
| `checkout_datetime` | ATTENDANCE.checkout_time | **text → TIMESTAMP**, nullable (13,144 blank) |
| `workout_type` | ATTENDANCE.workout_type | text → enum (4) |
| `trainer_id` | ATTENDANCE.trainer_id | text → FK, nullable (96,099 blank; populated only on Personal Training) |
| `duration_minutes` | ATTENDANCE.duration_minutes | **text → NUMBER** (20–120) |

**Mapped: 37 of 38 fields. Dropped: 1 (`discount_applied`), with reason.**

---

## 8. Dropped files

| File | Rows | Reason |
|---|---|---|
| `app_engagement_events.csv` | 97,988 | In-app events. The proposal puts the mobile app **out of scope**; nothing in FitTrack reads them. |
| `streak_history_episodes.csv` | 40,202 | Workout-streak episodes. Superseded by `ATTENDANCE`, from which streaks are derivable. |

Both files remain in `data/raw/` so the download stays byte-identical (`checksums.txt`).

---

## 9. Result (9 tables)

| Table | Rows from source | Source |
|---|---|---|
| `MEMBER` | 17,500 | `members_master` |
| `BRANCH` | 40 | `members_master` (3NF split) |
| `MEMBERSHIP_PLANS` | 4 | `plan_type` distinct values + seeded durations/prices |
| `SUBSCRIPTION` | 45,088 | `subscription_renewal_records` (676 duplicates removed) |
| `PAYMENT` | 45,764 | `subscription_renewal_records` |
| `TRAINERS` | 50 | `trainer_id` distinct values (attributes seeded) |
| `ATTENDANCE` | 108,744 | `gym_checkin_workout_logs` |
| `ALERTS` | 0 | runtime — written by `prc_expiry_alerts` |
| `AUDIT_LOG` | 0 | runtime — written by `trg_audit_payments` |

`ATTENDANCE` is also the **associative table** that resolves the many-to-many between `MEMBER` and `TRAINERS` (a member trains with many trainers; a trainer coaches many members) and between `MEMBER` and `BRANCH`.

See `data_dictionary.md` for columns, keys and cardinality.
