# AnomX – Midterm Submission

**Program:** AnomX Mentorship Program  
**Mentor:** Sugandh Kumar  
**Weeks Covered:** Week 1–2 (Foundations), Week 3 (Event Generation), Week 4 (Feature Engineering & EDA)

---

## Repository Structure

```
anomx/
├── data/
│   ├── raw/
│   │   └── events.csv
│   └── processed/
│       └── features.csv
├── src/
│   ├── generate_events.py
│   └── feature_engineering.py
├── notebooks/
│   ├── week3_event_generation_analysis.ipynb
│   └── week4_eda.ipynb
└── README.md
```

---

## Week 1–2: Python, Git & Foundations

### Python Basics

I came into this program already familiar with Python fundamentals — variables, data types, functions, loops, conditionals, and basic file handling. Rather than revisiting syntax, I focused on applying these concepts directly to the codebase provided.

Key Python patterns I encountered and understood in this project:

- **Dictionaries and nested structures** — user profiles are stored as dictionaries keyed by `user_id`, each containing the user's country, device, typical trade volume, and other attributes
- **Functions with clear responsibilities** — the code is split into separate functions like `build_user_profiles()`, `generate_normal_event()`, and individual anomaly generators. Each does one thing and returns a result
- **`np.random` and `random` module** — used throughout for seeded, reproducible data generation. Setting a seed means the same dataset is produced every time the script runs
- **`datetime` and `timedelta`** — used to generate realistic timestamps and compute time differences between events
- **List building with loops** — events are generated one at a time and appended to a list, which is then converted into a DataFrame at the end

### Git & GitHub Workflow

Git is used to track changes to code over time, collaborate with others, and maintain a history of decisions. The core workflow used in this project:

```bash
git init                           # initialise repo
git add .                          # stage changes
git commit -m "meaningful message" # save a snapshot
git push origin main               # push to GitHub
```

Good commit messages describe what changed and why — not just "update" or "fix".

### Pandas and NumPy Basics

**Pandas** is used to load, filter, group, and analyze tabular data. The key operations used in this project:

```python
df["column"].value_counts()          # count occurrences
df[df["event_type"] == "trade"]      # filter rows
df.groupby("user_id")["col"].mean()  # aggregate per user
df["timestamp"] = pd.to_datetime()   # parse dates
```

**NumPy** is used for numerical operations — particularly `np.random.normal()` and `np.where()` for generating and transforming values.

### Understanding Anomaly Detection

Anomaly detection is the task of identifying data points that differ significantly from expected behavior. In financial systems this means finding users whose activity — trades, logins, deposits, withdrawals — doesn't match their own history or the platform's norms.

Two broad approaches exist:
- **Supervised** — requires labeled data where we already know which events are fraudulent. Our dataset provides `is_anomalous` labels, making this approach possible
- **Unsupervised** — detects outliers without any labels, using clustering or statistical thresholds

This project uses labeled synthetic data, which lets us verify whether our features actually separate normal from anomalous behavior before building a model.

### Project Overview and Objectives

AnomX is an end-to-end anomaly detection system built on a synthetic financial trading dataset. The goal across all weeks is:

1. Generate realistic synthetic event data that mimics a trading platform
2. Engineer behavioral features from raw event logs
3. Perform exploratory data analysis to understand what the data contains
4. Build machine learning models that can flag suspicious users automatically

Real financial data is restricted due to privacy and legal concerns. Synthetic data allows us to experiment safely while keeping the problem realistic.

### Documentation Practices

Good documentation means a reader can understand what code does without running it. Key practices used here:

- **Function docstrings** — short descriptions of what a function takes as input and what it returns
- **Inline comments** — for non-obvious logic, especially in loops and mathematical operations
- **README files** — explain the project structure, how to run scripts, and what each file does
- **Notebooks with markdown** — observations written after each analysis cell explain what the output means, not just what the code does

---

## Week 3: Event Generation

**Files:** `src/generate_events.py` | `data/raw/events.csv` | `notebooks/week3_event_generation_analysis.ipynb`

### How Synthetic Event Data Was Generated

The script `generate_events.py` follows a four-step pipeline:

**Step 1 — Build User Profiles (`build_user_profiles()`)**  
Each synthetic user is assigned stable baseline attributes: home country, home IP, preferred device, typical trade volume, and typical deposit amount. About 10% of users are marked as anomalous and assigned a specific fraud pattern.

**Step 2 — Generate Normal Events (`generate_normal_event()`)**  
For each event, a type is sampled from a weighted distribution (trade: ~35%, login: ~22%, deposit: ~15%, etc.). Attribute values are drawn from distributions centered on the user's profile, so each user has consistent but slightly varied behavior.

**Step 3 — Inject Anomaly Patterns**  
Anomalous users have dedicated generator functions that produce events with intentionally suspicious characteristics. These run alongside normal event generation to create a mixed, realistic dataset.

**Step 4 — Export to CSV**  
All events are sorted by timestamp and saved to `events.csv`.

### Types of Events Used

| Event Type | Description | Key Columns |
|---|---|---|
| `login` | User authentication | `ip_address`, `country`, `device`, `failed_attempts` |
| `trade` | Buying or selling an instrument | `trade_volume`, `pnl`, `instrument`, `trade_duration_seconds` |
| `deposit` | Funds added to account | `amount`, `method` |
| `withdrawal` | Funds removed from account | `amount`, `is_immediate_withdrawal` |
| `session` | Web or app session | `session_duration_mins`, `click_rate_per_min` |
| `kyc_change` | Identity document update | High-risk when followed by large withdrawals |

### Anomaly Types Implemented

| Anomaly Type | Simulated Behavior |
|---|---|
| `ip_hopper` | Rapid switching between countries and IPs — impossible travel |
| `wash_trader` | Extremely high volume, same instrument, always profitable |
| `deposit_withdrawal_cycler` | Repeated deposit → immediate withdrawal pattern |
| `bot_trader` | Very high click rates, short sessions, unusual hours |
| `structurer` | Many small deposits just below a detection threshold |
| `brute_forcer` | Multiple failed logins followed by a successful one |
| `dormant_withdrawer` | Long inactivity followed by a large withdrawal |
| `consistent_winner` | Always profitable, very short holding times |
| `device_switcher` | Different device fingerprint on every login |
| `kyc_manipulator` | KYC details changed just before a large withdrawal |

### Dataset Structure and Key Insights

The raw dataset contains **50,000 events across 500 users**. About **5.3% of events are anomalous** — this class imbalance is intentional and realistic, since fraud is rare in production systems.

Key findings from `week3_event_generation_analysis.ipynb`:

- Trade events are the most frequent, followed by logins. KYC changes are the rarest, which reflects realistic platform usage
- Anomalous trades have significantly higher volumes and greater variability than normal trades — visible in a boxplot comparison
- Anomalous sessions have much higher click rates than normal sessions, consistent with automated or bot-like behavior
- Anomalous withdrawals are larger on average, driven by `dormant_withdrawer` and `kyc_manipulator` patterns
- Login failure rates are higher for anomalous users, particularly brute forcers who repeatedly fail before succeeding

---

## Week 4: Feature Engineering & EDA

**Files:** `src/feature_engineering.py` | `data/processed/features.csv` | `notebooks/week4_eda.ipynb`

### Features Created from Raw Event Logs

The script `feature_engineering.py` adds 20+ new columns across six categories:

**1. Time-Delta Features**
```
time_since_last_event_sec
time_since_last_login_sec
time_since_last_deposit_sec
```
Capture how quickly a user acts after their last event. Sudden activity after long dormancy is a known signal for compromised accounts.

**2. Rolling Window Features**
```
roll_5_trade_vol_mean,  roll_10_trade_vol_mean
roll_5_trade_vol_std,   roll_10_trade_vol_std
roll_5_pnl_mean,        roll_10_pnl_mean
roll_5_click_rate_mean, roll_10_click_rate_mean
```
Summarize recent behavior across windows of 5 and 10 events. A user suddenly trading 10x their rolling average stands out clearly.

**3. Burst Count Features**
```
burst_count_5min
burst_count_30min
```
Count events triggered within the last 5 or 30 minutes. Normal users rarely exceed 1 event per 5-minute window.

**4. Login Behavior Features**
```
unique_ips_last_10_logins
unique_countries_last_10_logins
unique_devices_last_10_logins
rolling_failed_attempts_5
```
Capture consistency in authentication. Switching countries repeatedly is a strong signal for `ip_hopper`. Repeated failed attempts flag `brute_forcer`.

**5. Financial Features**
```
roll_5_deposit_sum
withdrawal_to_deposit_ratio
```
Measures how much a user withdraws relative to their recent deposits. A ratio of 76 — withdrawing 76x more than deposited — is what `kyc_manipulator` users produce.

> **Bug found and fixed:** The original script divided by `roll_5_deposit_sum + 1e-9`, which caused ratios of ~2 trillion when a user withdrew before making any deposits. Fixed by only computing the ratio when `roll_5_deposit_sum > 100`.

**6. Z-Score Features**
```
trade_vol_zscore
pnl_zscore
amount_zscore
session_duration_zscore
```
Measure how far each event deviates from that user's own historical average. Personalized thresholds work better than global ones — a trade of ₹50,000 is suspicious for one user but normal for another.

### Why These Features Are Useful for Anomaly Detection

Raw events tell us *what happened*. Engineered features tell us *whether it's unusual*. A machine learning model cannot learn behavioral patterns from raw event logs — it needs numerical context about each user's recent history.

No single feature catches every fraud type. Each anomaly has a distinct signature:

| Anomaly Type | Most Discriminative Features |
|---|---|
| `ip_hopper` | `unique_countries_last_10_logins`, `unique_ips_last_10_logins` |
| `wash_trader` | `trade_vol_zscore`, `roll_5_trade_vol_mean`, `pnl_zscore` |
| `brute_forcer` | `rolling_failed_attempts_5`, `burst_count_5min` |
| `bot_trader` | `click_rate_per_min`, `burst_count_5min` |
| `kyc_manipulator` | `withdrawal_to_deposit_ratio`, `amount_zscore` |
| `dormant_withdrawer` | `time_since_last_login_sec`, `amount_zscore` |
| `device_switcher` | `unique_devices_last_10_logins` |

### EDA Performed

All analysis is in `notebooks/week4_eda.ipynb`.

**Task 1 — Event Distribution**  
Trade events dominate (~35%), KYC changes are the rarest (~6%). The dataset is not balanced by event type, which is realistic and needs to be accounted for during modeling.

**Task 2 — Temporal Behavior**  
Both time features are heavily right-skewed. The median gap between events is ~52,258 seconds (~14.5 hours), while the median gap between logins is ~216,786 seconds (~2.5 days) — users perform multiple actions between login sessions. Extreme inactivity gaps signal dormant account patterns.

**Task 3 — Burst Activity**  
49,702 out of 50,000 events have a burst count of exactly 1 in a 5-minute window. Only 12 events exceed a burst count of 3, and every single one belongs to either a `brute_forcer` or `ip_hopper`. Brute forcers fire rapid login attempts; IP hoppers log in from multiple locations almost simultaneously.

**Task 4 — Login Behavior**  
`ip_hopper` has a mean unique countries value of 0.90 vs 0.22 for normal users — the clearest single-feature separation in the dataset. Most other anomaly types have nearly identical login values to normal users, showing that login features alone cannot detect all fraud types.

**Task 5 — Financial Behavior**  
After fixing the feature engineering bug, the top withdrawal ratios correctly show `kyc_manipulator` (ratio up to 76x) and `dormant_withdrawer` (largest raw amount: ₹44,109) at the top. The `structurer` pattern is subtler — many small deposits of ~₹1,000 deliberately kept below thresholds.

**Task 6 — Z-Score Analysis**  
`trade_vol_zscore` has the most extreme values — 738 events exceed ±2, mainly `wash_trader` and `consistent_winner` users. `session_duration_zscore` has very few extremes (41 exceed ±2, none exceed ±3), confirming session durations are consistent per user and less useful for outlier detection.

**Task 7 — Correlation Analysis**  
`trade_volume` has the strongest correlation with `is_anomalous` (~0.22). The three login uniqueness features are very highly correlated with each other (~0.95–1.00), meaning they carry redundant information — one could likely be dropped before modeling.

### Challenge Task — Behavioral Profiles

**USER_0000 — Wash Trader**  
Average trade volume of 51,729, consistently positive PnL of +860.8, and a maximum `trade_vol_zscore` of 2.47. Login behavior is completely normal. This user would be flagged by financial and z-score features, but not by login features — showing why multiple feature types are necessary.

**USER_0002 — Device Switcher**  
Trading activity and PnL are entirely within normal range. The only signal is logging in from 5 different devices in the last 10 logins. A system relying only on financial features would miss this user completely.

**USER_0025 — Normal User**  
Stable trade volume (~27,612), slightly negative average PnL (−20.8, realistic — most traders don't consistently profit), single country, single device, low burst counts. One trade briefly exceeded z-score 2 but was an isolated event, not a pattern. Would not be flagged.

### Key Observations and Learnings

1. **No single feature is sufficient.** Each anomaly type leaves a different footprint. Financial, behavioral, temporal, and login features all need to work together.

2. **Feature engineering bugs can mislead a model completely.** The division-by-near-zero bug made normal users appear as the most suspicious people in the dataset. Catching it before modeling was critical.

3. **Z-scores are powerful because they are personalized.** A static threshold misses fraud from high-volume users and over-flags low-volume users. Computing deviation from each user's own history fixes this.

4. **Class imbalance must be addressed.** Only 5.3% of events are anomalous. A model predicting "normal" for everything would be 94.7% accurate but completely useless. Precision, recall, and F1-score matter more than accuracy here.

5. **EDA should always come before modeling.** The analysis revealed which features carry signal, which are redundant, and where bugs exist — all before writing a single line of model code.
