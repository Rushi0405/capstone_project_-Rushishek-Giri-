# Are Returns Eating Mamaearth's Margins?

An end-to-end SQL → Python → GenAI pipeline that finds which orders get returned, measures how much they cost, and turns the verified numbers into a plain-English brief for Mamaearth's ops and finance heads.

---

## Overview

**The problem.** Mamaearth's Growth Analytics team suspected that returns were cancelling out the revenue from some orders, but nobody had traced it from raw data to a decision. This project does that, end to end.

**The dataset.** Three CSV files covering January–June 2026:

| File | Rows | What it holds |
|---|---|---|
| `data/customers.csv` | 45 | Customer, city, city tier (1 or 2), signup date, acquisition source |
| `data/products.csv` | 16 | Product, category, price in rupees |
| `data/orders.csv` | 180 | Order date, quantity, discount %, payment method, rating, returned (1/0) |

The raw data has deliberate quality problems:
- missing discounts and ratings
- payment methods typed seven different ways
- five double-submitted orders
- two unusually large bulk orders

**Objectives:**
- Find the true revenue once data errors are removed.
- Test whether Cash on Delivery (COD) orders are returned more often than prepaid ones.
- Find where the return problem is concentrated.
- Check whether discounts reduce returns.
- Identify the real best month for revenue.
- Give the ops and finance heads a short, actionable brief.

**One rule throughout:** every number comes from code. Nothing is typed in by hand: not in the charts, the JSON or the AI prompt.

---

## Project Workflow

1. **Data Collection.** The three CSV files in `data/` come from the project brief. They are never edited by hand.
2. **Data Cleaning** (`analysis/clean_and_eda.py`):
   - **Payment method:** standardised the casing, so 7 spellings became 3 values (CARD 70, UPI 55, COD 55).
   - **Duplicates:** removed 5 double-submitted orders (O0176–O0180), matched on every column except `order_id`. This leaves 175 orders.
   - **Missing values:** filled *after* removing duplicates. A missing discount became 0 (12 rows) and a missing rating became the median, 3.0 (15 rows).
   - **Outliers:** flagged, not deleted, using the IQR rule. O0011 (25 units) and O0098 (30 units).
   - **Saved:** the cleaned tables go to `analysis/cleaned_data/` and the key results to `narrator/findings.json`.
3. **Exploratory Data Analysis (EDA):**
   - Cleaned revenue is **₹97,358.30**. The ₹2,501.90 gap to the raw total is exactly the 5 duplicates.
   - COD return rate is **44.4%**, against 18.9% for UPI and 14.7% for Card. The hypothesis is **confirmed**.
   - The worst segment is **COD in Tier-2 cities, at 54.5%** (Tier-1 COD is 37.5%).
   - All correlations are negligible. "Higher discounts reduce returns" is **busted** (r = −0.09).
   - With the outliers excluded, **March (₹20,318.90)** is the real peak. January only looked best because of the two bulk orders.
4. **Visualization** (`analysis/visualize.py`). It reads the cleaned CSV and draws two matplotlib charts:
   - a sorted bar chart of return rate by payment method, with COD highlighted;
   - a line chart of monthly revenue with the outliers excluded, with the peak labelled.

   Chart titles are built from the data.

   ![Return rate by payment method](visualizations/return_rate_by_payment.png)
   ![Monthly revenue, outliers excluded](visualizations/monthly_revenue_trend.png)
5. **SQL Analysis** (`sql/`):
   - `schema.sql` builds three linked tables with keys and constraints.
   - `seed_data.sql` loads every row, with blank cells stored as `NULL`.
   - `reports.sql` runs 9 business reports on the raw data, with each output pasted above its query.
6. **GenAI Narrative** (`narrator/generate_narrative.py`):
   - **Brief:** Gemini turns `findings.json` into a Situation–Complication–Resolution brief.
   - **Prompt design:** a system instruction sets the role, the structure and the rule "use only the supplied numbers". The prompt itself is built from `findings.json`.
   - **Settings:** temperature 0.0, a 2048-token limit and a 30-second timeout, all inside `try/except`.
   - **Offline fallback:** a template builds the same brief when there's no API key.
   - **Number check:** a checker confirms the 5 key figures are present and prints PASS or FAIL for each.

---

## Architecture

```
data/*.csv (raw)
   │
   ├──▶ SQL (SQLite):  schema.sql → seed_data.sql → reports.sql
   │                   Report (a) raw revenue = ₹99,860.20 ──────────┐ compared
   │                                                                  │
   └──▶ Python (pandas): analysis/clean_and_eda.py  ◀────────────────┘ prints MATCH
           │   cleans, analyses, recomputes the raw total
           ├──▶ analysis/cleaned_data/*.csv ──▶ analysis/visualize.py ──▶ visualizations/*.png
           └──▶ narrator/findings.json ──▶ narrator/generate_narrative.py ──▶ SCR brief
                                              (Gemini, or offline template)  + 5-figure check
```

- **SQL database for structured queries.** SQLite stores the raw orders and answers the 9 report questions. As the brief requires, the Python layer reads the CSVs directly, so the two layers can run in either order. Python recalculates SQL's raw total to prove they agree, and prints `MATCH` if the database exists.
- **Python scripts for cleaning and EDA.** `clean_and_eda.py` does every cleaning and analysis step and prints each result. It hands off two outputs: the cleaned CSVs for the charts, and `findings.json` for the narrative.
- **Visualizations for insights.** `visualize.py` only reads the cleaned CSV, so it never repeats the analysis.
- **GenAI for the business brief.** `generate_narrative.py` reads only `findings.json`. If the data changed, the brief would change with it.

---

## Tools & Libraries Used

| Tool | Version | Used for |
|---|---|---|
| **Python** | 3.9 or newer (tested on 3.13) | Cleaning, analysis, charts, narrative |
| **pandas** | ≥ 2.0 (tested on 3.0) | Loading, cleaning, merging, grouping, correlation, time series |
| **matplotlib** | ≥ 3.7 (tested on 3.11) | The two charts |
| **google-genai** | ≥ 1.0 (tested on 2.29) | Gemini API client; model `gemini-3.1-flash-lite` (free tier) |
| **SQLite** (`sqlite3` command-line tool) | 3.x (tested on 3.45) | Database, schema and the 9 SQL reports |
| **Python standard library** | — | `sqlite3` (cross-check with SQL), `json` (findings file), `os`, `re`, `datetime` |
| **Google Colab / Jupyter Notebook** | — | Running and testing the code interactively |
| **Git and GitHub** | — | Version control and submission |

numpy is installed automatically with pandas but isn't used directly. seaborn isn't used.

---

## AI Assistance Disclosure

I used AI tools (Claude, and Gemini through its API) on this project. This is what they were used for:

- **Data and SQL:** converting the raw data into CSV files, writing `schema.sql` and `seed_data.sql`, and writing the output 'reports.sql'.
- **Python:** writing the cleaning code,I asked for simple, well-commented code so that I could follow and explain each step.
- **GenAI layer:** writing the Gemini narrator, offline fallback and number checker, following the patterns from my class Gemini notebook. Gemini itself writes the business brief in `narrator/sample_output.txt`.
- **Checking and debugging:** re-running the pipeline to confirm every number matched the brief. This caught issues such as integer division in SQL and numbers typed in by hand.
- **Documentation:** drafting this README, including the Reflections, which were based on the points I chose.

**My part:** I provided the brief and the raw data, ran all the code myself in Google Colab, generated `sample_output.txt` with my own free Gemini key, and checked the results against the brief.

---

## Folder Structure

```
capstone_project_-Rushishek-Giri-/
├── README.md
├── requirements.txt              Python packages
├── data/                         raw CSVs (never edited)
│   ├── customers.csv
│   ├── products.csv
│   └── orders.csv
├── sql/
│   ├── schema.sql                Part 1, Task 1: tables, keys, constraints
│   ├── seed_data.sql             Part 1, Task 2: loads all rows (blanks → NULL)
│   └── reports.sql               Part 1, Task 3: 9 reports with outputs pasted above
├── analysis/
│   ├── clean_and_eda.py          Part 2, Tasks 1–10; saves cleaned CSVs + findings.json
│   ├── visualize.py              Part 2, Task 11: draws the 2 charts from the cleaned CSV
│   └── cleaned_data/             created when clean_and_eda.py runs
├── visualizations/
│   ├── return_rate_by_payment.png
│   └── monthly_revenue_trend.png
└── narrator/
    ├── findings.json             verified numbers, written by clean_and_eda.py
    ├── generate_narrative.py     Part 3: Gemini brief + offline fallback + checker
    └── sample_output.txt         one real Gemini run, saved and checked
```

---

## How to Run the Project

Run every command from the main project folder, in this order. Results are identical on every run.

**1. Get the project and install the dependencies**
```bash
git clone https://github.com/Rushi0405/capstone_project_-Rushishek-Giri-.git
cd capstone_project_-Rushishek-Giri-
pip install -r requirements.txt
```
You also need the `sqlite3` command-line tool:
- **Mac:** usually pre-installed.
- **Linux:** `sudo apt-get install sqlite3`
- **Windows:** download the sqlite-tools bundle from the official SQLite site.
- **Google Colab:** `!apt-get -qq install -y sqlite3`

On Windows, use Command Prompt, not PowerShell, for the `<` commands below.

**2. Load the SQL schema and data, then run the reports (Part 1)**
```bash
sqlite3 mamaearth.db < sql/schema.sql
sqlite3 mamaearth.db < sql/seed_data.sql
sqlite3 -header -column mamaearth.db < sql/reports.sql
```
Report (a) should print `180  99860.20  554.78`. Every report matches the `-- Output:` comment above it. Report (i) adds a column, so rebuild the database before running `reports.sql` again.

**3. Run the data cleaning and EDA (Part 2)**
```bash
python analysis/clean_and_eda.py
```
- **Prints** every task's result, including `Rs 97358.30`, `CONFIRMED`, `COD + Tier-2 cities at 54.5%`, `Highest month: 2026-03`, and `MATCH` if step 2 ran first.
- **Saves** the cleaned CSVs in `analysis/cleaned_data/`, plus `narrator/findings.json`.

**4. Create the charts**
```bash
python analysis/visualize.py
```
This reads `analysis/cleaned_data/orders_merged_clean.csv` and saves both PNGs in `visualizations/`.

**5. Generate the business narrative (Part 3)**
- **Without an API key** (offline template): `python narrator/generate_narrative.py`
- **With a free Gemini key from Google AI Studio:** set it as an environment variable, never in a file:
  ```bash
  export GEMINI_API_KEY="your-key-here"        # Mac / Linux
  set GEMINI_API_KEY=your-key-here             # Windows Command Prompt
  python narrator/generate_narrative.py
  ```
  In Google Colab, store the key in **Secrets**, then run:
  ```python
  import os
  from google.colab import userdata
  os.environ["GEMINI_API_KEY"] = userdata.get("SY_API_KEY")
  ```

Either way, you should see a Situation / Complication / Resolution brief followed by five `[PASS]` lines. If the key is wrong or the free quota runs out, the script automatically uses the offline brief instead of crashing.

---

## Key Findings

- **Revenue was overstated.** Five duplicate orders inflated the raw total (₹99,860.20). True revenue is **₹97,358.30**, and the ₹2,501.90 gap is exactly those duplicates.
- **COD is the return problem.** COD orders are returned **44.4%** of the time, about three times the card rate (14.7%). UPI is at 18.9%.
- **The damage is concentrated in Tier-2 cities.** **54.5%** of COD orders there come back, compared with 37.5% in Tier-1.
- **Discounts aren't the answer.** Discount and returns are barely related (r = −0.09).
- **March, not January, is the best month.** Two bulk orders made January look like the peak (₹29,582.10). Without them, **March, at ₹20,318.90**, leads.

**Recommendation:** confirm COD orders in Tier-2 cities before dispatch and encourage prepaid payment. Fix the double-submit bug at checkout. Report revenue only after removing duplicates.

---

## Reflections

**What surprised me most in the results?**

Two findings caught me off guard. The first came from splitting COD orders by city tier. I already knew COD had the worst return rate at 44.4%, but the difference between tiers was much bigger than I’d guessed. Tier-2 cities sent back 12 of their 22 COD orders, which is 54.5%. Tier-1 cities returned 37.5% of their 32. Looking only at the overall COD number would have hidden the fact that most of the problem sits in the smaller cities.

The second was January. On the first pass it looked like the strongest month by a wide margin, with ₹29,582.10 in revenue. When I looked closer, two very large orders were doing almost all the lifting: O0011, 25 units on 28 January, and O0098, 30 units on 10 January. Once those are set aside, January drops to ₹11,637.10 and March, at ₹20,318.90, turns out to be the actual best month. Had I built the monthly trend without first checking for outliers, I would have named the wrong month.

**Which step was hardest, and how did I solve it?**

The trickiest problems were the silent ones, where nothing crashes and the output still looks reasonable. SQLite treats discount_pct / 100 as whole-number division, so a 10% discount turns into 10/100 = 0. Every discount quietly vanishes and revenue is overstated, yet the query runs without complaint. Changing the divisor to 100.0 solved it.

The second problem was a hardcoded number. In my first version of the Python script, I had typed in the Part 1 total of 99,860.20 directly. That breaks the project rules, so I rewrote the script to work out the total on its own from the 180 raw rows. It still lands on the same figure as the SQL report, and the script now prints MATCH after comparing the two. The lesson from both was that an answer can look right and still be wrong. That’s why I added checks so each layer is compared against the one before it.

**What would I look at next with more time or data?**

The first thing I’d want is the reason behind each return. Right now the data only records whether an order came back (1 or 0), not why. If most COD returns turn out to be customers refusing the parcel at the door, confirming orders before dispatch or rewarding prepaid payment would make sense. If the main cause is damaged products, the focus should be on packaging and delivery instead. Knowing the reason would show Mamaearth where to act first.

The second is return rates at the product and category level. My analysis broke revenue down by category, but not returns. A handful of products could be coming back far more often than everything else. If so, fixing those would be quicker and cheaper than tackling an entire payment method.

---

## Author

**Rushishek Giri**

## Last Updated

19 October 2026
