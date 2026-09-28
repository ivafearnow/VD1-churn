# Results – V&D Project 1: Which Customers Should the Retention Team Contact?


**Environment:** Python 3.9.6 (course reference is 3.11); numpy 1.24.2, pandas 2.0.0, scikit-learn 1.2.2 — package versions match `VD1_requirements.txt` (see `outputs/compare_run.json`).
**Data:** `churn.csv` (instructor copy of the Telco churn example, CC BY 4.0, via the `scikit-learn/churn-prediction` Hugging Face mirror), 7,043 rows × 21 columns, 1,869 Yes / 5,174 No. SHA-256 `16320c9c…5e91`.
**Commands run:**
```
python VD1_analysis.py compare  --csv churn.csv --out outputs
python VD1_analysis.py evaluate --csv churn.csv --out outputs --choice trees
```

---

## Validation choice (recorded before running `evaluate`)

**Date/time recorded:** Sep 27, 2026, 10:18 PM MT — before the `evaluate` run at 10:19 PM (timestamped in `agent_dialog.md`, Codex Prompt 4)
**Chosen method:** trees

| Method | Validation AUC | Top-20% churn rate (281 contacted) | Overall churn rate |
|---|---|---|---|
| Contract rule | 0.743 | 41.6% | 26.5% |
| Logistic regression | 0.838 | 61.2% | 26.5% |
| Boosted trees | 0.846 | 65.1% | 26.5% |

**Reason:** My reason is because it has the highest AUC (0.846) and the top -20 top-20% list had the most actual churners (65.1%, vs 61.2% for logistic and 41.6% for the contract rule). across the 3 models, meaning that fewer waseted calls. on people who were going to stay anyway.


---

## 1. Data cleaning decisions

| Decision | What was done | Why |
|---|---|---|
| Missing values | 11 blank `TotalCharges`, all with `tenure = 0` → filled with 0; no rows removed | Eleven customers have a tenure of 0 months, meaning no total charges have been recorded yet. Removing these 11 customers would result in a loss of valuable information because they are real new customers. Replacing the missing values with 0 is more appropriate than using the average (~$2,283), which would incorrectly make brand-new customers appear to have accumulated charges similar to those of longer-tenured customers. Therefore, a good solution is to assign a value of 0 to their TotalCharges, preserving these customers in the dataset without introducing misleading information. |
| Outliers | None removed. Ranges: tenure 0–72 months, MonthlyCharges $18–$119, TotalCharges $0–$8,685 (right-skewed) | We retained the outliers because they represent valid customer behavior. The maximum TotalCharges of $8,685 is realistic given a tenure of up to 72 months and MonthlyCharges of $119 ($119 × 72 ≈ $8,568). Removing these observations could cause the model to lose valuable information about long-tenure customers and their churn patterns. |
| Standardize? | Yes — numeric inputs scaled with `StandardScaler`; categorical inputs one-hot encoded; both learned from training rows only (code lines 84–89) | We standardized the numeric inputs (tenure, MonthlyCharges, and TotalCharges) to put them on the same scale, since TotalCharges reaches thousands while tenure only goes up to 72 months. This helps logistic regression converge and makes its coefficients more comparable, while decision trees do not require standardization. We fitted the scaler using only the training data to prevent information from validation or test customers, such as their mean and standard deviation, from leaking into the model and making its performance appear better than it really is. |
| Inputs | Same 7 inputs for both models: tenure, MonthlyCharges, TotalCharges, Contract, InternetService, PaperlessBilling, PaymentMethod. `customerID` used only to check uniqueness | ✍️ (optional) |
| Separation | 4,225 train / 1,409 validation / 1,409 test (stratified, seed 0); 0 rows in more than one partition (`split_rows.csv`); fitting uses `train` rows only (lines 76–77, 89) | We checked that the 7,043 customers were split into training (4,225), validation (1,409), and test (1,409) groups, with 0 overlapping customers. This ensures that the model is evaluated on unseen customers, preventing data leakage and providing an honest estimate of its performance. |

---

## 2. Final-test AUC across three methods (1,409 test customers)

| Method | Test AUC | Top-20% churn rate (281 contacted) | Overall test churn rate |
|---|---|---|---|
| Contract rule (baseline) | 0.737 | 39.9% | 26.5% |
| Logistic regression | 0.847 | 69.4% | 26.5% |
| Boosted trees (chosen) | 0.850 | 69.4% | 26.5% |

**What AUC measures:** AUC measures how well a model ranks customers who churn above those who stay, rather than its prediction accuracy. Boosted trees (0.850) and logistic regression (0.847) both performed better than the contract rule baseline (0.737), with boosted trees having a slightly higher AUC.

**Devon's contact-list question:** The top-20% churn rate is the most relevant metric for Devon because it measures how many customers on the contact list actually churned. Both boosted trees and logistic regression identified 69.4% churners among the 281 customers, compared with 39.9% for the contract rule and the overall churn rate of 26.5%.

---

## 3. Uncertainty – 95% bootstrap intervals (1,000 paired resamples of the test rows)

| Comparison | Estimate | 95% interval | Crosses zero? |
|---|---|---|---|
| Contract rule AUC | 0.737 | 0.716 – 0.756 | — |
| Logistic AUC | 0.847 | 0.825 – 0.870 | — |
| Trees AUC | 0.850 | 0.828 – 0.873 | — |
| **Logistic − contract** | +0.110 | 0.093 – 0.129 | No |
| **Trees − contract** | +0.112 | 0.096 – 0.131 | No |
| Trees − logistic | +0.002 | −0.005 – 0.010 | Yes |

**What paired means and why it is used:** Paired bootstrap means both methods are evaluated on the same resampled customers. This allows us to compare their performance fairly by reducing the influence of differences in the customers selected.

**Logistic and trees vs. contract rule:** The 95% confidence intervals for logistic regression (+0.093 to +0.129) and boosted trees (+0.096 to +0.131) are entirely above zero. This provides evidence that both models rank customers better than the contract rule baseline.

**Trees vs. logistic regression:** The confidence interval for trees vs. logistic regression (−0.005 to +0.010) crosses zero, meaning we cannot confidently determine which model performs better. Although we selected boosted trees, the results do not establish that they outperform logistic regression.

**One limitation of these intervals:** These intervals only measure uncertainty from the test customers sampled. They do not account for future changes in customer behavior, which could affect the model's performance over time.

---

## Supporting checks (used in the memo)

### Probability accuracy – chosen method (trees)

| | n | Mean predicted | Observed churn |
|---|---|---|---|
| Overall | 1,409 | 27.0% | 26.5% |
| Top-20% list | 281 | 64.8% | 69.4% |
| Group 0.0–0.2 | 724 | 8.0% | 7.2% |
| Group 0.2–0.4 | 272 | 29.8% | 25.4% |
| Group 0.4–0.6 | 236 | 49.6% | 52.1% |
| Group 0.6–0.8 | 142 | 67.7% | 69.0% |
| Group 0.8–1.0 | 35 | 83.4% | 91.4% |

Boosted trees are well calibrated overall, with 27.0% predicted churn compared to 26.5% observed churn. However, the top-20% contact list slightly underpredicts churn (64.8% vs. 69.4%). The 0.8–1.0 risk group has only 35 customers, making its observed churn rate less reliable due to the small sample size.

### Value scenarios – trees top-20% list (r = 69.4%)

| Assumed save rate (s) | Net value per 1,000 contacts |
|---|---|
| 10% | −$1,620 |
| 15% | +$670 |
| 20% | +$2,960 |

Break-even save rate (trees) = $6.20 / (0.694 × $66) = **13.5%**  ·  Contract rule: 23.6%, negative at all three save rates.

**Independent arithmetic check (done by hand, trees, s = 15%):**
1. 0.694 × 0.15 = 0.1041
2. 0.1041 × $66 = $6.8706
3. $6.8706 − $6.20 = $0.6706
4. $0.6706 × 1,000 = **$670.6** → matches `scenarios.csv` (+$670.11; the small difference is from rounding r to 0.694)
