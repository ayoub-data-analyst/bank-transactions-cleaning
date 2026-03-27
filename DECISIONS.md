# DECISIONS.md — Cleaning Decisions Log

## Project: FinanceCore SA — Banking Transactions Pipeline

**Source:** `bank_transactions.csv` (2,060 rows × 16 columns)

**Output:** `financecore_clean.csv`

**Notebook:** `main.ipynb`

---

## STEP 1 — Import & Exploration

### D-01 · Initial dataset inspection

| Field                                    | Detail                                                                                                                                                                                                                                                           |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Observations**                   | Loaded with `pd.read_csv()`, inspected via `shape`,`dtypes`,`head()`,`describe()`                                                                                                                                                                      |
| **Problematic columns identified** | `date_transaction`(mixed formats),`montant`(comma decimal),`solde_avant`(text suffix),`devise`(mixed case),`segment_client`(mixed case + NaN),`agence`(whitespace + NaN),`score_credit_client`(out-of-range values + NaN),`taux_interet`(unused) |
| **Decision**                       | No changes at this stage — diagnostic phase only                                                                                                                                                                                                                |

### D-02 · Missing value rate

| Field                      | Detail                                                     |
| -------------------------- | ---------------------------------------------------------- |
| **Method**           | `df.isna().sum()`then `(missing / len(df)) * 100`      |
| **Visualization**    | Matplotlib bar chart per column                            |
| **Affected columns** | `segment_client`,`agence`,`score_credit_client`      |
| **Decision**         | Treatment deferred to steps 2 and 3 based on variable type |

### D-03 · Duplicate detection

| Field              | Detail                                                            |
| ------------------ | ----------------------------------------------------------------- |
| **Method**   | `df[df.duplicated(subset=["transaction_id"], keep=False)]`      |
| **Result**   | Rows sharing the same `transaction_id`with near-identical dates |
| **Decision** | Removal handled in step 2                                         |

---

## STEP 2 — Data Cleaning

### D-04 · Drop column `taux_interet`

| Field                   | Detail                                                                      |
| ----------------------- | --------------------------------------------------------------------------- |
| **Problem**       | Column present in the source file but not required by any pipeline analysis |
| **Decision**      | **Dropped**via `df.drop(columns=["taux_interet"])`                  |
| **Justification** | Reduces dataset noise; no pipeline step references it                       |

### D-05 · Deduplication on `transaction_id`

| Field                   | Detail                                                                                                                     |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Problem**       | Duplicate transactions: same `transaction_id`, near-identical dates — likely an artefact of the central system export   |
| **Decision**      | **Keep the first occurrence**(`keep="first"`), drop subsequent duplicates                                          |
| **Justification** | The first occurrence is treated as the original transaction; duplicates result from repeated exports or system sync errors |
| **Code**          | `df = df.drop_duplicates(subset=["transaction_id"], keep="first")`                                                       |

### D-06 · Date format unification (`date_transaction`)

| Field                     | Detail                                                                                                                                                   |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Problem**         | Two coexisting formats:`DD/MM/YYYY`and `YYYY-MM-DD`                                                                                                  |
| **Decision**        | Conversion via `pd.to_datetime(errors="coerce")`→ unified format `YYYY-MM-DD HH:MM:SS`                                                              |
| **Unparsable rows** | **Dropped**(`dropna(subset=["date_transaction"])`)                                                                                               |
| **Justification**   | A transaction with no valid date is unusable for time-series analysis;`errors="coerce"`produces `NaT`for unrecognized formats, which are then purged |
| **Code**            | `df["date_transaction"] = pd.to_datetime(df["date_transaction"], errors="coerce")`+`df = df.dropna(subset=["date_transaction"])`                     |

### D-07 · Decimal separator fix (`montant`)

| Field                   | Detail                                                                                                                  |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **Problem**       | Amounts encoded with a comma decimal separator (e.g.`"1234,56"`) — column typed as `object`, not directly castable |
| **Decision**      | Replace `,`→`.`then cast to `float`                                                                              |
| **Justification** | Python/Pandas requires a period as the standard decimal separator                                                       |
| **Code**          | `df["montant"] = df["montant"].str.replace(",", ".").astype(float)`                                                   |

### D-08 · Remove text suffix in `solde_avant`

| Field                   | Detail                                                                                                                           |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Problem**       | Values like `"12500.0 EUR"`— the text suffix prevents numeric conversion                                                      |
| **Decision**      | Strip the `"EUR"`string and trailing whitespace, then cast to `float`                                                        |
| **Justification** | The currency is already captured in the dedicated `devise`column; the suffix is redundant and incompatible with numeric typing |
| **Code**          | `df["solde_avant"] = df["solde_avant"].str.replace("EUR", "").str.strip().astype(float)`                                       |

### D-09 · Case normalization (`devise`)

| Field                   | Detail                                                                                   |
| ----------------------- | ---------------------------------------------------------------------------------------- |
| **Problem**       | Mixed values:`"eur"`,`"Eur"`,`"EUR"`— creates separate groups during aggregations |
| **Decision**      | Convert to**uppercase**(`str.upper()`)                                           |
| **Justification** | ISO 4217 standard: currency codes are conventionally uppercase                           |
| **Code**          | `df["devise"] = df["devise"].str.upper()`                                              |

### D-10 · Case harmonization (`segment_client`)

| Field                   | Detail                                                                                         |
| ----------------------- | ---------------------------------------------------------------------------------------------- |
| **Problem**       | Inconsistent values:`"PREMIUM"`,`"premium"`,`"Premium"`coexist                           |
| **Decision**      | Normalize to**capitalize**(`str.lower().str.capitalize()`)                             |
| **Justification** | Human-readable format for the dashboard; prevents segment fragmentation in GROUP BY operations |
| **Code**          | `df["segment_client"] = df["segment_client"].str.lower().str.capitalize()`                   |

### D-11 · Missing value imputation (`segment_client`)

| Field                   | Detail                                                                                                                                             |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Problem**       | `NaN`values in the column after case normalization                                                                                               |
| **Decision**      | **Mode imputation**(most frequent value)                                                                                                     |
| **Justification** | Nominal categorical variable — median does not apply; mode preserves the existing segment distribution without introducing an artificial category |
| **Code**          | `df["segment_client"] = df["segment_client"].fillna(df["segment_client"].mode()[0])`                                                             |

### D-12 · Strip whitespace (`agence`)

| Field              | Detail                                                                                                      |
| ------------------ | ----------------------------------------------------------------------------------------------------------- |
| **Problem**  | Leading/trailing spaces (e.g.`" Casablanca "`) creating false duplicates during branch-level aggregations |
| **Decision** | Apply `str.strip()`                                                                                       |
| **Code**     | `df["agence"] = df["agence"].str.strip()`                                                                 |

### D-13 · Missing value imputation (`agence`)

| Field                   | Detail                                                                                                                         |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Problem**       | Residual `NaN`values after stripping                                                                                         |
| **Decision**      | **Mode imputation**                                                                                                      |
| **Justification** | Categorical variable — same reasoning as `segment_client`; assigning the most frequent branch minimizes the introduced bias |
| **Code**          | `df["agence"] = df["agence"].fillna(df["agence"].mode()[0])`                                                                 |

---

## STEP 3 — Outlier Detection & Treatment

### D-14 · Out-of-range credit scores (`score_credit_client`)

| Field                   | Detail                                                                                                                                                                         |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Problem**       | Scores < 300 or > 850 — outside the standard FICO range; likely data entry or transfer errors                                                                                 |
| **Method**        | Business rule: fixed thresholds [300, 850]                                                                                                                                     |
| **Decision**      | **Nullify**out-of-range values →**impute with median**                                                                                                            |
| **Justification** | The median is robust to residual extremes and preserves the central distribution; dropping the entire row would be too aggressive (loss of otherwise valid client information) |
| **Code**          | `df.loc[(df["score_credit_client"] < 300) \| (df["score_credit_client"] > 850), "score_credit_client"] = None`then `fillna(median_score)`                                   |

### D-15 · Outlier amounts (`montant`)

| Field                   | Detail                                                                                                                                                                                                                                                                                  |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Problem**       | Extreme amounts present: €0, +€99,999, −€200,000                                                                                                                                                                                                                                    |
| **Method**        | **IQR** :`lower = Q1 − 1.5×IQR`,`upper = Q3 + 1.5×IQR`                                                                                                                                                                                                                     |
| **Decision**      | **Keep with flag**→ boolean column `is_anomaly_montant`                                                                                                                                                                                                                        |
| **Justification** | Extreme amounts may be legitimate (international wire transfer, accounting correction, refund). Dropping without business validation is too risky. The flag allows the Risk Management team to audit these transactions separately in the dashboard without altering the global dataset |
| **Code**          | `df["is_anomaly_montant"] = (df["montant"] < lower) \| (df["montant"] > upper)`                                                                                                                                                                                                        |

---

## STEP 4 — Feature Engineering

### D-16 · Temporal decomposition (`date_transaction`)

| New column              | Extraction                                                                              |
| ----------------------- | --------------------------------------------------------------------------------------- |
| `anne`                | `dt.year`                                                                             |
| `mois`                | `dt.month`                                                                            |
| `trimestre`           | `dt.quarter`                                                                          |
| `jour_semaine`        | `dt.day_name()`                                                                       |
| **Justification** | Essential for seasonality analysis, monthly trends, and activity peaks in the dashboard |

### D-17 · EUR conversion verification

| Field                   | Detail                                                                                                                          |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **Column**        | `montant_eur_verifie = montant / taux_change_eur`                                                                             |
| **Gap**           | `ecart_eur = montant_eur − montant_eur_verifie`                                                                              |
| **Flag**          | `is_conversion_error = True`if `                                                                                              |
| **Justification** | Detects inconsistencies in exchange rates applied by the source system — a data quality indicator for the Risk Management team |

### D-18 · Credit risk category (`categorie_risque`)

| Score                   | Category                                                                                                                                           |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| ≥ 700                  | `Low`                                                                                                                                            |
| 580 – 699              | `Medium`                                                                                                                                         |
| < 580                   | `High`                                                                                                                                           |
| **Justification** | Segmentation inspired by FICO scoring adapted to FinanceCore banking practices; used directly by the Risk Management team for portfolio monitoring |

### D-19 · Net balance per client (`solde_net`)

| Field                   | Detail                                                                                               |
| ----------------------- | ---------------------------------------------------------------------------------------------------- |
| **Logic**         | Credits = amounts > 0; Debits = amounts < 0;`solde_net = Σ credits − Σ debits`per `client_id` |
| **Integration**   | Left merge on `client_id`to bring `solde_net`back into the main DataFrame                        |
| **Justification** | Financial health indicator per client; required for segmentation in the dashboard                    |

### D-20 · Behavioral aggregations per client

| Column                    | Calculation                                                                        |
| ------------------------- | ---------------------------------------------------------------------------------- |
| `nb_transactions`       | `count(transaction_id)`per `client_id`                                         |
| `montant_moyen`         | `mean(montant)`per `client_id`                                                 |
| `nb_produits_distincts` | `nunique(produit)`per `client_id`                                              |
| **Justification**   | KPIs required for behavioral client scoring and segmentation — General Management |

### D-21 · Rejection rate per branch (`taux_rejet`)

| Field                   | Detail                                                                                              |
| ----------------------- | --------------------------------------------------------------------------------------------------- |
| **Formula**       | `(nb "Rejete" transactions / total transactions) × 100`per `agence`                            |
| **Justification** | Key operational performance indicator — enables identification of branches with high failure rates |

---

## STEP 5 — Export

| Decision                | Detail                                                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Format**        | UTF-8 CSV, comma separator, no index (`index=False`)                                                              |
| **File**          | `financecore_clean.csv`                                                                                           |
| **Quality check** | No missing values on critical columns; correct numeric types on `montant`,`solde_avant`,`score_credit_client` |

---

*Document produced as part of the FinanceCore SA project — Data Engineering Module, Week 02.*
